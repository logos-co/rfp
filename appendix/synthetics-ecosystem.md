# Appendix: CDP Synthetics Ecosystem

Reference material for
[RFP-024: CDP-Backed Synthetic Assets](../RFPs/RFP-024-cdp-backed-synthetic.md).
It covers protocols that issue several synthetics from one system, the stability
pool CDPs whose liquidation design RFP-024 uses, and how these protocols price
assets, charge fees and pay for liquidations.

Figures were read on 2026-11-16 unless stated otherwise. TVL and peaks are from
the DefiLlama API (`https://api.llama.fi/protocol/<slug>`); on-chain values were
read from the protocols' contracts.

## Designs

| Design           | Who carries a synthetic's risk                                                                  | Liquidation                             | Examples                                            |
| ---------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------- | --------------------------------------------------- |
| Debt pool        | All stakers together: each staker's debt is a share of the value of every synthetic outstanding | Per staker, on the staker's own c-ratio | Synthetix V1/V2                                     |
| Per-position CDP | The borrower: each position's debt is what it minted                                            | Per position, on its own c-ratio        | Liquity, Sky/Maker, Indigo, Youves, Metronome Synth |

RFP-024 asks for per-position CDPs.

## Protocols issuing several synthetics

| Protocol        | Chain              | Synthetics                    | Collateral                                | What joins the synthetics                       | Liquidation                                                  | TVL (peak)                   | Status                                                                           |
| --------------- | ------------------ | ----------------------------- | ----------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------ | ---------------------------- | -------------------------------------------------------------------------------- |
| Synthetix V1/V2 | Ethereum, Optimism | sUSD, sETH, sBTC and others   | SNX only                                  | One debt pool shared by all stakers             | Flag, then forced liquidation with a penalty paid to stakers | ~$0 ($2.97B, 2021-02-10)     | Legacy: solo staking retired April 2025                                          |
| Indigo          | Cardano            | iUSD, iBTC, iETH, iSOL        | ADA and whitelisted Cardano native tokens | A stability pool per synthetic                  | Position frozen, then absorbed by the stability pool; 2% fee | $2.28M ($125.6M, 2024-03-12) | Live (V3)                                                                        |
| Youves          | Tezos              | uUSD, uBTC, uXTZ              | XTZ, tzBTC, USDt, SIRS                    | An engine contract per synthetic and collateral | Partial step-in by a liquidator, 112.5% payout               | $16.2M                       | Live                                                                             |
| Metronome Synth | Ethereum, OP, Base | msUSD, msETH, msBTC           | USDC, DAI, FRAX, ETH, WBTC, vault tokens  | Cross-margin positions and a shared swap pool   | Partial or full, 18% fee (10% liquidator, 8% protocol)       | $7.89M ($33.0M, 2025-10-07)  | Impaired: about $15.7M unbacked disclosed 2026-07-30, swaps paused, msUSD $0.818 |
| Mirror          | Terra              | mAssets (equities and others) | UST, aUST, LUNA, MIR, ANC                 | One mint contract for every mAsset              | Discounted auction                                           | $0 ($2.21B, 2021-05-12)      | Dead since 2022                                                                  |

Synthetix V2 issued `sETH`, `sBTC` and, from the Hadar release (SIP-49,
2020-03-30), `sXMR`. `sXMR` has since been removed: it is no longer registered
in the Synthetix issuer contract. A proposal to add `sZEC` (SIP-119, February
2021\) was rejected.

Metronome's shortfall came from its swap module, which minted and burned
synthetics at oracle price against a shared pool: about 6,367 msETH and 4.57M
msUSD were created without adequate collateral, which the DAO attributed to
oracle latency on Base.

## Stability pool CDPs

These issue a single USD stablecoin rather than several synthetics, but use the
stability pool liquidation that RFP-024 requires.

| Protocol                   | Chain    | Collateral                   | Stability pool and liquidation                                               | TVL (peak)                   |
| -------------------------- | -------- | ---------------------------- | ---------------------------------------------------------------------------- | ---------------------------- |
| Liquity V1 (LUSD)          | Ethereum | ETH                          | Stability pool, then redistribution to other positions; minimum c-ratio 110% | $130.3M ($4.52B, 2021-05-11) |
| Liquity V2 (BOLD)          | Ethereum | ETH, wstETH, rETH            | A stability pool per collateral, 5% penalty; redistribution fallback         | $70.7M ($180.0M, 2025-10-07) |
| Felix (feUSD)              | HyperEVM | HYPE, UBTC, kHYPE            | Liquity V2 design, 5% penalty                                                | $17.8M ($294.1M, 2025-07-14) |
| Mezo (MUSD)                | Mezo     | BTC                          | Stability pool; minimum c-ratio 110%                                         | $1.10M (MUSD supply $18.86M) |
| Sky/Maker (for comparison) | Ethereum | Several, per collateral type | No stability pool: Dutch auction of the collateral                           | $3.76B ($19.82B, 2021-12-01) |

## Price feeds and staleness

Protocols that issue several synthetics price every asset, collateral and
synthetic alike, against one quote currency (USD, or UST for Mirror), with one
feed per asset. The c-ratio uses the ratio of the two USD prices.

| Protocol     | Staleness rule                                                                                                                          |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| Synthetix V2 | Rates older than 25 hours (`rateStalePeriod`), or flagged or circuit-broken, block issuance                                             |
| Mirror       | Prices expire after 60 seconds; operations fail on an expired price                                                                     |
| Liquity V1   | 4 hour timeout per oracle; Chainlink treated as broken on a jump over 50% between rounds; Tellor as fallback; otherwise last good price |
| Liquity V2   | Staleness threshold per feed (24 hours for ETH/USD); on failure the branch shuts down and uses the last good price                      |
| Sky/Maker    | Prices pass through a 1 hour delay (OSM) before use                                                                                     |

In each multi-synthetic protocol, the protocol's own unit of account is fixed at
1 rather than read from a feed: sUSD in Synthetix, UST in Mirror and msUSD in
Metronome (1.0 on chain while it traded at $0.818). A depeg of that unit never
reaches the c-ratio check.

## Fees and liquidation incentives

| Protocol        | Borrower pays                                                                     | Liquidation caller receives                                                                                                  | Stability pool or backers receive                                                                              | Protocol revenue goes to                                   |
| --------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Liquity V1      | One-off fee of 0.5% to 5%, rising with redemptions; no interest                   | 200 LUSD reserved when the position opens, plus 0.5% of its collateral                                                       | Liquidated collateral at a discount (about 10% at the 110% minimum c-ratio)                                    | LQTY stakers                                               |
| Liquity V2      | Interest at a rate the borrower sets; upfront fee of 7 days' average interest     | 0.0375 WETH reserved when the position opens (refunded on close), plus the lesser of 0.5% of the collateral or 2 units of it | Liquidated collateral with a 5% penalty; 75% of interest                                                       | The other 25% of interest, to DEX liquidity incentives     |
| Mezo            | 1% to 5% a year, fixed at opening; 0.1% issuance fee                              | 200 MUSD gas compensation plus 0.5% (Liquity V1 style)                                                                       | Liquidated collateral                                                                                          | [DATA NEEDED]                                              |
| Sky/Maker       | Stability fee per collateral type (ETH-A 10.50% a year on chain)                  | Flat DAI tip plus a chip proportional to the vault's debt                                                                    | No stability pool; penalty per collateral type                                                                 | Savings rates first, then SKY buybacks and staking rewards |
| Synthetix V2    | No interest; exchange fees between synthetics (35 bps for sETH and sBTC on L1)    | 10 SNX for flagging an account and 20 SNX for liquidating it (L1)                                                            | Penalty SNX shared among stakers: 30% forced, 20% self per SIP-148; 60% and 50%, with 180 day escrow, on chain | Stakers                                                    |
| Indigo          | Mint fee 0.1% (API) or 0.5% (docs); algorithmic interest                          | [DATA NEEDED]                                                                                                                | Liquidated collateral (2% liquidation fee, recipient [DATA NEEDED])                                            | [DATA NEEDED]                                              |
| Youves          | Mint fee 1.5625% (uUSD, uBTC) or 0.3906% (uXTZ); liability spread about 1% a year | Payout of 112.5% or 110%                                                                                                     | No stability pool                                                                                              | [DATA NEEDED]                                              |
| Metronome Synth | Swap fee 0.45% (0.55% on Ethereum)                                                | 10% liquidation fee                                                                                                          | n/a                                                                                                            | 8% liquidation fee and swap fees                           |

Liquity V2 redistribution (when the stability pool cannot cover a position)
takes a penalty of up to 10% on the WETH branch and 20% on the staked ETH
branches. Its WETH reserve "does not count as Trove's collateral": it backs no
debt and is left out of the c-ratio. The Liquity V2 README gives the stability
pool's share of interest as "72%, final value TBD"; the deployed constant
(`SP_YIELD_SPLIT`) is 75%.

In pool-backed perpetuals exchanges, the pool that backs positions takes a fixed
share of fees: 63% at GMX V2 and 75% at Jupiter Perps.

## Failure and survival

Across 50 CDP and debt-pool protocols surveyed, 20 had ended, halted or dropped
their CDP product by 2026-11-16 (by the survey's own status labels). Patterns
seen in the failures:

- **Collateral from the protocol's own ecosystem.** Mirror (UST and LUNA
  collateral), Kujira USK (leverage held by the Kujira Foundation) and Iron
  Finance (TITAN). Synthetix governance cut the target c-ratio from 800% to 350%
  to 400% as SNX fell.
- **Oracle-priced minting outside the liquidation path.** Metronome's swap
  module in July 2026.
- **Complex or upgradeable peripheral contracts.** Abracadabra, Prisma and Raft
  lost funds through contracts around the core.

Among survivors, Liquity V1 and V2 combine an immutable core, few collateral
types and redemption at face value. Hard redemption at par mostly held tighter
pegs than softer anchors on 2026-11-16 (LUSD $1.003, BOLD $0.9985; msUSD
$0.818), with exceptions such as Gravita (GRAI $0.4057 despite redemption).

## References

- [DefiLlama protocol API](https://api.llama.fi/protocols)
- [Liquity V1 repository](https://github.com/liquity/dev) and
  [borrowing FAQ](https://docs.liquity.org/liquity-v1/faq/borrowing.md)
- [Liquity V2 (BOLD) README](https://github.com/liquity/bold/blob/main/README.md)
  and [FAQ](https://docs.liquity.org/v2-faq/borrowing-and-liquidations.md)
- [Mezo fees](https://mezo.org/docs/users/borrow/fees.md)
- [Felix documentation](https://usefelix.gitbook.io/docs/lending-products/quickstart/faq.md)
- [Sky collateral liquidation](https://developers.skyeco.com/protocol/vaults/collateral-liquidation/)
- [SIP-148: Upgrade Liquidation Mechanism](https://sips.synthetix.io/sips/sip-148/)
- [SIP-49: Add new synths for Hadar release](https://sips.synthetix.io/sips/sip-49/)
- [SIP-119: Add ZEC crypto Synth](https://sips.synthetix.io/sips/sip-119/)
- [SIP-302: Pools V3](https://sips.synthetix.io/sips/sip-302/)
- [Indigo V3 documentation](https://docs.indigoprotocol.io/readme/indigo-protocol-v3.md)
- [Youves collateral management](https://docs.youves.com/syntheticAssets/stableTokens/collateralManagement/Collateral-Management-Details)
- [Metronome revenue model](https://docs.metronome.io/metronome-synth/metronome-revenue-model)
- [Metronome DAO oracle shortfall (Crypto Briefing)](https://cryptobriefing.com/metronome-dao-16m-oracle-shortfall/)
- [GMX fees](https://docs.gmx.io/docs/trading/fees/)
- [Jupiter Perps fees and JLP (Eco)](https://eco.com/support/en/articles/15083164-jupiter-perps-fees-leverage-how-jlp-works)
