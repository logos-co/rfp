# Appendix: Synthetic Assets Ecosystem

Reference material for
[RFP-024: CDP-Backed Synthetic Assets](../RFPs/RFP-024-cdp-backed-synthetic.md).
It surveys on-chain protocols that give synthetic price exposure across Ethereum
and its L2s, Solana, app-chains and Bitcoin L2s, then covers the price feed, fee
and liquidation designs relevant to a CDP synthetic protocol.

The survey data was accessed in May 2026. The fee and liquidation data for
stability pool CDPs, and the second TVL table, were accessed on 2026-11-16.

## Protocols surveyed

| Protocol      | Ecosystem                             | Design                                  | Metric        | Value              | Date    | Source                                                                                                                          |
| ------------- | ------------------------------------- | --------------------------------------- | ------------- | ------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Hyperliquid   | Own L1 (USDC bridge on Arbitrum)      | Orderbook perpetuals                    | TVL           | $5.11B             | 2026-05 | [DefiLlama](https://defillama.com/protocol/hyperliquid)                                                                         |
| Jupiter Perps | Solana                                | LP-pool perpetuals, oracle priced       | JLP pool TVL  | ~$1.3B             | 2026-04 | [Eco](https://eco.com/support/en/articles/15083164-jupiter-perps-fees-leverage-how-jlp-works)                                   |
| sBTC          | Stacks (Bitcoin L2)                   | 1:1 BTC peg held by threshold signers   | TVL           | $437M (peak $545M) | 2026 Q1 | [Stacks](https://www.stacks.co/blog/q1-2026-snapshot)                                                                           |
| Drift         | Solana                                | Hybrid orderbook and vAMM perpetuals    | TVL           | $150M to $400M     | 2026    | [Eco](https://eco.com/support/en/articles/15083167-drift-protocol-solana-perpetuals-dex-deep-dive)                              |
| GMX V2        | Arbitrum, Avalanche, MegaETH, Botanix | LP-pool perpetuals, oracle priced       | TVL           | $203.47M           | 2026-05 | [DefiLlama](https://defillama.com/protocol/gmx-v2-perps)                                                                        |
| dYdX v4       | dYdX Chain (Cosmos)                   | Orderbook perpetuals                    | Open interest | ~$175M             | 2026    | [Coinbureau](https://coinbureau.com/review/dydx)                                                                                |
| Ostium        | Arbitrum                              | LP-pool perpetuals, including RWA pairs | Open interest | $213M+             | 2026-04 | [MEXC](https://blog.mexc.com/ostium-airdrop-2026-how-to-farm-points-on-the-rwa-perp-dex-bringing-gold-oil-and-stocks-on-chain/) |
| Synthetix V3  | Ethereum, Base, Arbitrum              | CDP synthetic tokens and perpetuals     | TVL           | $41.19M            | 2026-05 | [DefiLlama](https://defillama.com/protocol/synthetix-v3)                                                                        |

Synthetix is the only CDP synthetic token protocol in the survey. The table
below adds other CDP synthetic protocols listed on DefiLlama, and updates
Synthetix.

| Protocol        | Ecosystem                | TVL (2026-11-16)                   | Peak TVL            |
| --------------- | ------------------------ | ---------------------------------- | ------------------- |
| Synthetix v1+v2 | Ethereum, Optimism       | ~$0 (last above $1k on 2025-08-08) | $2.97B (2021-02-10) |
| Synthetix V3    | Ethereum, Arbitrum, Base | ~$0                                | $390M (2024-12-09)  |
| Mirror          | Terra                    | ~$0 (last above $1k on 2023-01-19) | $2.21B (2021-05-12) |
| Youves          | Tezos                    | $16.2M                             | $43.7M (2024-03-13) |
| Metronome Synth | Ethereum, Optimism, Base | $7.9M                              | $33.0M (2025-10-07) |

Haven Protocol (xAssets minted against XHV on its own Monero-fork L1) is not
listed on DefiLlama. It closed on 2024-12-12 after a range-proof validation flaw
let at least 1.3 billion XHV be minted illicitly.

## Synthetix

Synthetix V1 and V2 used a single debt pool: SNX stakers minted synths (`sUSD`,
`sBTC`, `sETH` and others) against staked SNX, typically at a c-ratio above
500%, and shared the debt of all synths between them. Synthetix listed `sETH`,
`sBTC` and `sXMR` from this one engine, each through a price feed and a set of
parameters (Hadar release, 2020-03-30). `sXMR` was priced by Chainlink feeds. A
proposal to add `sZEC` (SIP-119, February 2021) was rejected.

V3 replaced the single pool with pools that back specific markets and accept
several collateral types. SIP-302 describes it: "The Vault Module creates vaults
of each collateral type per pool, similar to a CDP (MakerDAO, Liquity), where
accounts can delegate their staked collateral to pools by opening staking
positions and then mint and burn snxUSD, backed by their collateral." Stakers
choose which markets their pool backs, so they choose which debt they carry.

In December 2025 Synthetix moved V3 back to Ethereum, with Base as its growth
target, and began winding down its Arbitrum perpetuals. In May 2026 governance
committed all trading fee revenue to SNX and sUSD buybacks.

## Price feeds

Every oracle-priced protocol in the survey uses Chainlink Data Streams, Pyth, or
both:

- **Chainlink Data Streams:** pull-based signed price reports. Used by GMX V2
  (only source), Synthetix V3 (one input) and Ostium (crypto pairs).
- **Pyth:** pull-based oracle. Used by Drift (primary), Jupiter Perps (one of
  three feeds) and Synthetix V3 (one input).
- **Edge by Chaos Labs:** used by Jupiter Perps as one of its three feeds.
- **Custom aggregator:** Ostium aggregates licensed exchange data for gold, oil,
  forex and equities.

Aggregation across feeds:

- Synthetix V3's Oracle Manager lets each market combine feeds, for example the
  minimum of Chainlink, Pyth and a Uniswap TWAP.
- Jupiter Perps aggregates three feeds, so the protocol can fall back if one
  deteriorates.
- Drift configures a fallback per market.
- GMX V2 and Synthetix V3 settle orders against the next oracle update after the
  order is submitted, so the executor cannot pick a stale price.

## Fees and incentives

### Pool-backed protocols

| Protocol      | Who backs positions | Backers' fee share      | Other fee destination                 |
| ------------- | ------------------- | ----------------------- | ------------------------------------- |
| Jupiter Perps | JLP pool            | 75% of platform fees    | Protocol, remaining 25%               |
| GMX V2        | GM and GLV pools    | 60% to 70% of fees      | [DATA NEEDED]                         |
| Synthetix V3  | Staked debt pools   | A share of trading fees | SNX and sUSD buybacks (from May 2026) |
| Hyperliquid   | HLP vault           | A share of fees         | [DATA NEEDED]                         |

In each of these, backers also absorb losses when the positions they back win.
Hyperliquid's HLP vault performs liquidations as well as making markets.

### Stability pool CDPs

These issue a USD stablecoin, not a synthetic, but use the stability pool and
liquidation design that RFP-024 requires.

| Protocol            | Ecosystem | TVL (2026-11-16) | Peak TVL            |
| ------------------- | --------- | ---------------- | ------------------- |
| Liquity V1 (LUSD)   | Ethereum  | $130.3M          | $4.52B (2021-05-11) |
| Liquity V2 (BOLD)   | Ethereum  | $70.7M           | $180M (2025-10-07)  |
| Sky Lending (Maker) | Ethereum  | $3.76B           | $19.8B (2021-12-01) |

| Protocol     | Borrower pays                                          | Liquidation caller receives                                                                                                       | Stability pool or backers receive                                                    | Protocol revenue goes to                   |
| ------------ | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------ |
| Liquity V1   | 0.5% one-time borrowing fee                            | 200 LUSD reserved when the position opens, plus 0.5% of the position's collateral                                                 | Liquidated collateral at a discount (about 10% at the 110% minimum c-ratio)          | LQTY stakers                               |
| Liquity V2   | Upfront fee, then interest at a rate the borrower sets | 0.0375 WETH reserved when the position opens (refunded on close), plus the lesser of 0.5% of the collateral or 2 collateral units | Liquidated collateral (penalty at most 10%), plus a fixed share of interest          | Remainder of interest to DEX LP incentives |
| Sky/MakerDAO | Stability fee (variable, set by governance)            | Flat DAI tip plus a chip proportional to the vault's debt (Liquidations 2.0)                                                      | No stability pool; collateral is sold by Dutch auction                               | Surplus buffer, MKR buy-and-burn           |
| Synthetix v2 | [DATA NEEDED]                                          | Flat SNX reward for flagging an account (10 SNX on L1, 1 on L2) and for liquidating it (20 SNX on L1, 2 on L2)                    | Penalty SNX (30% forced, 20% self-liquidation) shared among stakers, 12-month escrow | Stakers                                    |

Notes:

- Liquity V2's share of interest paid to the stability pool is given as "72%,
  final value TBD" in the BOLD README and as 75% in the
  [Liquidation & Auction Ecosystem appendix](./appendix-liquidation-auction-ecosystem.md).
- In Liquity V2 the WETH reserve "does not count as Trove's collateral": it
  backs no debt and is left out of the c-ratio.
- Liquity V2 uses separate penalties for stability pool offsets and for
  redistribution, with
  `LIQUIDATION_PENALTY_SP <= LIQUIDATION_PENALTY_REDISTRIBUTION <= 10%`.
- Metronome Synth and Youves fee schedules: [DATA NEEDED].

The
[Liquidation & Auction Ecosystem appendix](./appendix-liquidation-auction-ecosystem.md)
and the
[Reflexive Stablecoin Ecosystem appendix](./appendix-reflexive-stablecoin-ecosystem.md)
cover these mechanisms in more detail.

## References

Survey (accessed May 2026):

- [DefiLlama: Synthetix V3](https://defillama.com/protocol/synthetix-v3)
- [DefiLlama: Synthetix v1+v2](https://defillama.com/protocol/synthetix-v1+v2)
- [Synthetix: What is Synthetix V3](https://blog.synthetix.io/what-is-synthetix-v3/)
- [Synthetix V3 oracle documentation](https://docs.synthetix.io/synthetix-protocol/the-synthetix-protocol/oracles)
- [The Defiant: Synthetix to pivot V3 perps from Arbitrum to Base](https://thedefiant.io/news/defi/synthetix-to-pivot-v3-perps-from-arbitrum-to-base)
- [Synthetix sXMR token (Etherscan)](https://etherscan.io/token/0x5299d6F7472DCc137D7f3C4BcfBBB514BaBF341A)
- [SIP-119: Add ZEC crypto Synth](https://sips.synthetix.io/sips/sip-119)
- [SIP-302: Pools V3](https://sips.synthetix.io/sips/sip-302)
- [Synthetix: New Synths update for the upcoming Hadar release](https://blog.synthetix.io/new-synths-update-for-the-upcoming-hadar-release/)
- [DefiLlama: Hyperliquid](https://defillama.com/protocol/hyperliquid)
- [DefiLlama: GMX V2 Perps](https://defillama.com/protocol/gmx-v2-perps)
- [Eco: Jupiter Perps fees, leverage and JLP](https://eco.com/support/en/articles/15083164-jupiter-perps-fees-leverage-how-jlp-works)
- [Jupiter support: fees](https://support.jup.ag/hc/en-us/articles/18735045234588-Fees)
- [Eco: Drift Protocol deep dive](https://eco.com/support/en/articles/15083167-drift-protocol-solana-perpetuals-dex-deep-dive)
- [Coinbureau: dYdX review](https://coinbureau.com/review/dydx)
- [MEXC: Ostium](https://blog.mexc.com/ostium-airdrop-2026-how-to-farm-points-on-the-rwa-perp-dex-bringing-gold-oil-and-stocks-on-chain/)
- [Stacks: Q1 2026 snapshot](https://www.stacks.co/blog/q1-2026-snapshot)
- [Haven Protocol: Project Closure Announcement (2024-12-12)](https://havenprotocol.org/2024/12/12/project-closure-announcement/)

Fees, liquidation and CDP TVL (accessed 2026-11-16):

- [DefiLlama protocol API](https://api.llama.fi/protocols)
- [Liquity V2 (BOLD) README](https://github.com/liquity/bold/blob/main/README.md)
- [Liquity v1 repository](https://github.com/liquity/dev)
- [SIP-148: Upgrade Liquidation Mechanism](https://sips.synthetix.io/sips/sip-148/)
- [Metronome Synth documentation](https://docs.metronome.io/metronome-synth/metronome-synth-protocol.md)
