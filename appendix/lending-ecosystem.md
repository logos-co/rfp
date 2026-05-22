---
title: Lending and Borrowing Ecosystem
status: draft
updated: 2026-05-19
---

# Appendix: Lending and Borrowing Ecosystem

This appendix surveys decentralised lending and borrowing protocols
across the Ethereum, Solana, broader Web3 (BNB Chain, Avalanche,
Tron, Sui, Aptos), and Bitcoin ecosystems. It serves as a reference
for [RFP-008](../RFPs/RFP-008-lending-borrowing-protocol.md) and
[RFP-012](../RFPs/RFP-012-curated-lending-vaults.md), with a deep
dive on Morpho Blue as the priority design referenced by both RFPs.

The figures below are snapshots between 2026-04 and 2026-05. Source
URLs are cited inline; cumulative-loss data is drawn from public
post-mortems published between 2020 and 2026.

## Ecosystem Landscape (May 2026)

The DeFi lending category holds approximately **$54B in TVL** across
**380+ active protocols on 80+ chains**, with the top ten capturing
~78% of deposits. Lending recovered its 2021 dollar peak by mid-2025
and has since reshaped around four structural axes:

- **Ethereum** is bifurcated. The pooled monoliths
  ([Aave](https://aave.com), [Compound](https://compound.finance))
  continue to absorb the majority of consumer flow. A parallel
  *primitive + curator* stack
  ([Morpho Blue](https://morpho.org) +
  [MetaMorpho](https://docs.morpho.org/learn/concepts/vault-v2/))
  absorbs institutional and B2B flow that wants narrower,
  deterministic risk.
- **Solana** consolidated around [Kamino](https://app.kamino.finance)
  (~60% lending market share), with
  [Jupiter Lend](https://jup.ag/lend) mounting a fast (and
  contested) challenge. MarginFi receded after its 2024 operational
  crisis; Save (formerly Solend) is the OG that did not recapture
  momentum.
- **Bitcoin** matured from "wrapped BTC on EVM" into four distinct
  families (LST bridge, DLC P2P, ICP threshold signing, Rootstock
  sidechain), with [Babylon](https://babylonlabs.io) anchoring the
  underlying staking yield and
  [Lombard](https://lombard.finance) dominating the LST surface.
- **Long-tail chains** (BNB Chain, Avalanche, Tron, Cosmos, Sui,
  Aptos) each have a dominant lender, but none has meaningfully
  challenged the EVM mainstream or Solana for new innovation.

## Protocols Surveyed

Protocols are ranked by TVL within the DeFi lending category (top 20),
followed by a Bitcoin-native sub-table. The order is preserved
throughout the document.

| Rank | Protocol | Ecosystem | TVL (approx.) | Lending Model | Website |
|------|----------|-----------|---------------|---------------|---------|
| 1 | Aave | Ethereum + multichain | $34-44B (all versions); $19.4B V3 alone | Pooled (V3); modular hub-and-spoke (V4) | [aave.com](https://aave.com) |
| 2 | Morpho Blue (full stack) | Ethereum + Base + others | $11.78B | Isolated markets singleton + curated vaults | [morpho.org](https://morpho.org) |
| 3 | Spark | Ethereum + Base | $5-8.5B | Pooled (Aave V3 fork; Sky-allocated USDS) | [spark.fi](https://spark.fi) |
| 4 | Babylon | Bitcoin | $5B staked | Native BTC staking (not lending per se) | [babylonlabs.io](https://babylonlabs.io) |
| 5 | Kamino | Solana | $3.2-3.5B | Isolated markets (V2 modular primitive) | [app.kamino.finance](https://app.kamino.finance) |
| 6 | Compound | Ethereum + multichain | $2.7B (V3) + $1.4B legacy | Pooled (V2); single-borrow isolated collateral (V3 "Comet") | [compound.finance](https://compound.finance) |
| 7 | JustLend | Tron | $2.4B | Pooled (Compound V2 fork) | [justlend.org](https://justlend.org) |
| 8 | Maple | Ethereum + Solana | ~$2.1B AUM | Pool-delegate undercollateralised; KYC + permissionless syrupUSDC wrapper | [maple.finance](https://maple.finance) |
| 9 | Lombard | Bitcoin (LST) | $1.9B | LST issuer (LBTC) | [lombard.finance](https://lombard.finance) |
| 10 | Jupiter Lend | Solana | ~$1.65B | Vaults with rehypothecation | [jup.ag](https://jup.ag) |
| 11 | Fluid (Instadapp) | Ethereum | $1.6B | Lending + DEX hybrid (smart collateral / smart debt) | [fluid.io](https://fluid.io) |
| 12 | Venus | BNB Chain | $1.47-1.64B | Pooled (Compound V2 fork) | [venus.io](https://venus.io) |
| 13 | Euler V2 | Ethereum + Base | $890M | Modular vault kit (EVK) + cross-vault composition (EVC) | [euler.finance](https://www.euler.finance) |
| 14 | Silo | Ethereum + Arbitrum + Avalanche + Sonic | $750M | Paired isolated silos (two-asset ERC-4626 vault pairs) | [silo.finance](https://silo.finance) |
| 15 | Save (formerly Solend) | Solana | $300M | Pooled + isolated pools | [save.finance](https://save.finance) |
| 16 | Benqi | Avalanche | $262M | Pooled | [benqi.fi](https://benqi.fi) |
| 17 | MarginFi | Solana | mid-tier post-incident | Pooled (banks) | [marginfi.com](https://marginfi.com) |
| 18 | Suilend | Sui | $146M | Pooled | [suilend.fi](https://suilend.fi) |
| 19 | Scallop | Sui | $23M | Pooled with PTB-native flash loans | [scallop.io](https://scallop.io) |
| 20 | Gearbox | Ethereum + Arbitrum + Optimism | $44M | Credit accounts (per-position smart contracts) | [gearbox.fi](https://gearbox.fi) |

**Bitcoin / BTCFi sub-table:**

| Protocol | Model | TVL / Volume |
|----------|-------|--------------|
| Babylon | Native BTC staking | $5B staked |
| Lombard | LST bridge (LBTC) | $1.9B |
| Liquidium | DLC P2P (Ordinals/Runes/BRC-20 collateral) | $450M cumulative loan volume; 102,000+ loans |
| Sovryn | Rootstock sidechain (pooled + Liquity-style Zero) | Tens of millions |
| Lava | DLC v2 + BLOC | $200M facility |

Sources: per-protocol notes link out to
[DefiLlama](https://defillama.com), Phemex, eco.com,
[PANews BTCFi report](https://www.panewslab.com/en/articles/0mmtz1k6),
The Block, Coindesk, DL News, and primary project documentation.

## Lending Models

The architecture space has converged on a small number of recurring
patterns. The list below captures the models observed across the
surveyed protocols.

### Pooled lending (Compound / Aave model)

Shared per-asset pools with rebasing or exchange-rate receipts
(cTokens, aTokens). Borrowers post other supplied assets as
collateral subject to per-asset LTV. Liquidations are permissionless
with a fixed bonus.

Used by Aave V1-V3, Compound V2 (and all V2 forks: JustLend, Venus,
Benqi, Radiant), Spark (Aave V3 fork), MarginFi (per-bank pools).
Still the deepest-liquidity venue but losing share to isolated
designs.

### Isolated markets

Each lending market is a self-contained tuple of
`(loanAsset, collateralAsset, oracle, IRM, LLTV)`. Bad debt or
oracle failure in one market cannot contaminate another. Risk is
local; capital efficiency can be tuned per market.

- **Morpho Blue**: every market is a tuple in the singleton contract;
  permissionless creation. The dominant 2025-2026 design.
- **Silo V2**: two-asset "silos" via paired ERC-4626 vaults;
  permissionless deployment.
- **Euler V2**: every market is a separately deployed vault, optionally
  composed via the Ethereum Vault Connector (EVC).
- **Kamino V2**: modular markets supporting peer-to-peer, RWA, and
  orderbook integrations.
- **Save**: isolated pools for long-tail tokens.
- **Aave** (limited): isolation mode for new asset listings,
  restricted to stablecoin borrowing with per-asset debt ceilings.

Stress tests during 2024-2025 LRT depegs and stablecoin wobbles
confirmed local containment for Morpho Blue and Silo markets.

### Single-borrow isolated collateral (Compound V3 / Comet)

Each Comet instance allows borrowing **one** base asset (USDC, WETH,
USDT) against a set of approved collateral assets that are siloed
from the borrowable pool. Collateral cannot itself be borrowed,
eliminating cross-asset contamination present in V2 and Aave. Per
[Compound's 2026 targets](https://volity.io/crypto/compound/), up to
50 Comet deployments are planned across chains.

### Curated vaults (the dominant distribution layer)

ERC-4626 (or equivalent) vaults that aggregate deposits in a single
loan asset and route them across a curator-chosen basket of
underlying isolated markets. The curator picks markets, sets
per-market supply caps, and adjusts allocation within constraints
(timelock, allocator role).

Canonical example:
[MetaMorpho](https://docs.morpho.org/learn/concepts/vault-v2/) V1
and V2 above Morpho Blue. Major curators in 2026: Gauntlet,
Steakhouse Financial, Re7, MEV Capital, Block Analitica, Apostro.
Curator fees typically 5-15% of yield.

MetaMorpho V2 (launched
[2025-09-30](https://morpho.org/blog/morpho-vaults-v2-a-new-standard-for-asset-curation/))
adds **ID-based risk caps** (cap total exposure to a risk class such
as a collateral asset, market, or protocol), configurable role
segregation for institutional separation-of-duties, and **in-kind
redemption** via flash loan with an optional exit penalty up to 2%.

Vaults are also the dominant abstraction in Kamino (Earn vaults) and
Euler V2 (EVK vaults).

### Credit accounts

Per-position smart contracts that enable composable leverage by
holding a borrower's collateral and debt in a contract whose
permissions are restricted to whitelisted adapter interactions.
Used by [Gearbox](https://gearbox.fi) ("Credit Accounts") and Mars
Protocol (Rovers).

### Lending + DEX hybrid

[Fluid](https://fluid.io) (Instadapp) unifies lending and DEX
liquidity: collateral supplied for borrowing is simultaneously
placed as DEX liquidity ("smart collateral"); debt earns from being
on the buy-side of the DEX ("smart debt"). The result is materially
higher capital efficiency than separate lending and DEX layers.

### Undercollateralised institutional credit

[Maple](https://maple.finance) runs KYC-gated pools where trading
firms and crypto-native funds borrow USDC from professional
underwriters. The `syrupUSDC` wrapper exposes these underlying loan
yields to permissionless deposits in eligible jurisdictions. Maple's
2026 ARR target is $100M, drawn from real borrower interest rather
than spread.

### Bitcoin-native lending (four families)

- **LST bridge**: Babylon-staked BTC → LSTs (LBTC, others) used as
  collateral on EVM lending venues. Dominant by integration count
  (Lombard's LBTC has 70+ DeFi integrations).
- **DLC P2P**: Liquidium, Lava. Collateral is locked in Discreet Log
  Contracts on Bitcoin until repayment; loans settle atomically via
  Bitcoin script. Liquidium's cumulative volume reached $450M across
  102,000+ loans by 2025.
- **ICP threshold signing**: Internet Computer Chain Fusion enables
  trustless cross-chain lending without wrapped tokens or
  centralised bridges, used by LiquidiumFi.
- **Sidechain peg**: Sovryn on Rootstock; pooled lending plus a
  Liquity-style Zero CDP.

## Risk Machinery

### Oracle architectures

Oracles decide who is solvent and who is liquidatable. Every major
lending breach to date has either been an oracle bug or had an
oracle component.

| Design | Examples | Trade-off |
|--------|----------|-----------|
| Single-oracle Chainlink-style | Aave, Compound, most Venus/Compound forks | Well-vetted, broad coverage; single failure mode if a feed deviates |
| TWAP from on-chain AMM | Various Uniswap V3 / Curve integrations | Raises cost of prolonged manipulation; liquidity-dependent |
| Oracle-agnostic per-market | Morpho Blue | No protocol-mandated vendor; vault curators bear oracle-vetting responsibility |
| Dual oracle (LTV vs liquidation) | Silo V2 | "Loose" oracle for LTV, "tighter" oracle for liquidation; reduces false-positive liquidations under jitter |
| Exchange-rate oracles for LST/LRT | Many venues since 2023 | Required to price wstETH/LBTC/mSOL etc.; manipulation risk under de-peg events |
| Scam Wick Protection | Kamino | Rejects outlier price spikes that would trigger manipulation-driven liquidations |

### Liquidation mechanisms

| Mechanism | Examples | Trade-off |
|-----------|----------|-----------|
| Fixed-bonus permissionless | Aave, Compound, Morpho Blue, Venus | Predictable; bonus paid out of collateral; tuning is hard |
| Dutch / dynamic auction | Kamino V2 | Discovers fair price; depends on fast block times |
| Soft / partial liquidation | Aave V3, Morpho Blue | Minimises borrower loss; more complex accounting |
| Stability Pool absorption | Sovryn Zero (Liquity fork) | Atomic, gas-cheap; pool depletion leaves protocol exposed |

### Interest rate models

Most protocols use **kinked utilisation curves** (Compound V2 and
its forks; Aave V3) with parameters set by governance per asset.
Morpho Blue's
[Adaptive Curve IRM](https://morpho.org/blog/introducing-the-adaptivecurveirm-efficient-and-autonomous/)
is the notable departure: an immutable, autonomous IRM that targets
90% utilisation by adjusting the curve cumulatively over time, with
no per-asset governance retuning. This aligns with Morpho's
immutable-core philosophy and is well suited to a portfolio of
heterogeneous markets (LRT, LST, RWA, meme assets) that do not share
an equilibrium utilisation.

### Immutable core

A protocol whose primary contract is deployed once and never
upgraded. Bug fixes require deployment of a new version and user
migration. Trades flexibility for elimination of governance attack
surface and parameter-change risk.

- **Morpho Blue**: ~600 LOC singleton, no proxy, no pause, no admin
  functions over user funds. Governance whitelists IRMs and LLTVs
  (a discrete grid such as 0.385, 0.625, 0.77, 0.86, 0.915, 0.945,
  0.965) but cannot touch deployed markets.
- **Liquity / Sovryn Zero**: classic immutable CDP primitive.

Immutability has become a competitive differentiator. The 2023-2024
period saw multiple losses from upgrade paths (Radiant $58M
multisig compromise; Compound $80M COMP-bug; numerous smaller proxy
compromises). Institutional users (Coinbase) have explicitly cited
Morpho's immutability as a reason for preferring it as a back-end
for regulated retail products.

## Capital Efficiency

- **Efficiency mode (e-mode)**: Aave V3 raises LTV to ~97% for
  correlated assets (stablecoins, LSTs).
- **Leverage looping**: repeated supply/borrow to amplify exposure.
  Productised as Kamino Multiply (one-click) and Mars HLS. Available
  manually on most lenders.
- **Rehypothecation**: reuse of deposited collateral elsewhere in the
  protocol (Jupiter Lend, Fluid). Controversial when conflated with
  "isolation" marketing; the
  [Jupiter Lend rehypothecation dispute](https://www.theblock.co/post/381602/jupiter-exec-acknowledges-zero-contagion-claim-was-not-100-correct-after-backlash-over-vault-design)
  in late 2025 is the most public example.
- **Liquid staking tokens as collateral**: LSTs (wstETH, weETH,
  cbBTC, LBTC, mSOL, JitoSOL) are the dominant collateral class
  across every major venue.

## Governance and Revenue

Lending fee revenue correlates to leverage cycles and stablecoin
demand. Protocols that have layered on additional fee surfaces
capture more value per unit of TVL than pure pooled lenders.

| Protocol | Approx. annualised fees | Note |
|----------|-------------------------|------|
| Aave | Hundreds of millions | Dominant share of DeFi lending fees |
| Morpho Blue (full stack) | ~$175M | April 2026 |
| Maple | $100M target ARR for 2026 | Borrower-paid loan interest |
| Compound | ~$17M annualised | Daily ~$47k; 2026-Q2 |
| Silo V2 | n/a (mechanism) | 50% protocol revenue paid to xSILO stakers in USDC |

**Blended supply yields** (typical 2026 ranges):

- Compound V3: ~2% pre-gas (low utilisation).
- Aave stablecoin pools: 4-8% depending on asset and market.
- Morpho Blue vault deposits: 3-9% depending on vault.
- Maple syrupUSDC: ~10-14% net of fees, drawn from real loan
  interest.
- Babylon BTC staking: 3-12% annualised, varies by BSN mix.

**Governance stablecoins** extend the revenue surface beyond
borrow-supply spread. Aave issues GHO (and Savings GHO wrapper);
Sky issues USDS through Spark's allocator vaults. The borrow
interest paid by users minting these stablecoins accrues to the
issuing protocol's treasury rather than to supply-side lenders.

## Architecture Comparison (Surveyed Projects)

| Project | Lending model | Market isolation | Governance over markets | Mutability | Oracle policy |
|---------|---------------|------------------|-------------------------|-----------|---------------|
| Aave | Pooled (V3); modular V4 hub-and-spoke | Isolation mode (limited) | Yes, full | Upgradeable proxy + timelock | Chainlink-anchored |
| Compound V3 | Single-borrow Comet | Per-Comet | Yes, per Comet | Upgradeable | Chainlink + Redstone |
| Morpho Blue | Isolated markets singleton | Strong | Whitelist of IRMs + LLTVs only | **Immutable core** | Per-market oracle (any) |
| Spark | Pooled (Aave V3 fork) | Limited | Sky governance | Upgradeable | Chainlink-anchored |
| Euler V2 | Modular vaults; cross-vault composition (EVC) | Strong (per vault) | Whitelisting; vault deployment permissionless | Permissionless deploy; core can upgrade | Per-vault oracle |
| Silo V2 | Paired isolated silos | Strong | Permissionless silo creation | Per-silo | Dual oracle (LTV / liquidation) |
| Fluid | Lending + DEX hybrid | Limited | Instadapp governance | Upgradeable | Hybrid |
| Gearbox | Credit accounts | Per account | Whitelisting of adapters | Upgradeable | External (Chainlink) |
| Maple | Pool delegate undercollateralised | Per pool | Pool delegate + DAO | Upgradeable | Off-chain attestation |
| Kamino V2 | Isolated markets | Strong | DAO + permissionless | Upgradeable | Scam Wick Protection + Pyth/Switchboard |
| MarginFi | Pooled (banks) | Per bank | DAO + team | Upgradeable | Pyth/Switchboard |
| Save | Pooled + isolated pools | Mixed | Solend DAO | Upgradeable | Pyth/Switchboard |
| Jupiter Lend | Vaults with rehypothecation | Partial (contested) | Jupiter governance | Upgradeable | Pyth |
| JustLend | Pooled (Compound V2 fork) | None | Tron DAO | Upgradeable | Chainlink-equivalent |
| Venus | Pooled (Compound V2 fork) | None | XVS governance | Upgradeable | Chainlink |

Quick reads:

- Only Morpho Blue has an explicitly immutable core at scale.
- Permissionless market creation is now the norm for new protocols
  (Morpho Blue, Silo V2, Euler V2, Kamino V2, Scallop).
- Pooled lending remains dominant by TVL globally (Aave + every
  Compound V2 fork) but is losing share to isolated markets +
  curated vaults.

## Morpho Blue: Priority Deep Dive

Morpho Blue is the most consequential lending protocol to study as
of mid-2026 for three reasons:

1. **Architectural minimalism.** A ~600-LOC immutable singleton with
   permissionless market creation, formally verified, that pushes
   risk management out of governance and into a competitive curator
   market. See [the launch
   blog](https://morpho.org/blog/morpho-blue-and-how-it-enables-our-vision-for-defi-lending/)
   and the
   [whitepaper](https://resources.cryptocompare.com/asset-management/17952/1732199021661.pdf).
2. **Institutional distribution.** Coinbase routes
   [$1B+ of cbBTC-backed loans](https://www.theblock.co/post/373032/coinbase-tops-1-billion-in-bitcoin-backed-onchain-loans-via-morpho)
   and multi-billion USDC earn through Morpho. This is not abstract
   "DeFi growth": the largest US crypto exchange has embedded a
   smart-contract lending venue as the back-end for its regulated
   retail product. See the
   [Coinbase story](https://morpho.org/stories/coinbase/).
3. **Structural ceiling.** As of May 2026, Morpho's full stack TVL
   ($11.78B) and active loans (~$4B) put it second only to Aave.
   Network-effect compounding via curator-driven vault distribution
   suggests further share gains.

### Morpho Blue metrics

| Metric | Value | Date | Source |
|--------|-------|------|--------|
| Total Morpho TVL (full stack) | $11.78B | 2026-05-12 | [Phemex / DefiLlama summary](https://phemex.com/news/article/morpho-tvl-reaches-1178b-secures-second-place-in-defi-lending-80656) |
| Morpho Blue TVL (core) | ~$4.9B across 200+ markets (Ethereum + Base) | 2026-04 | [eco.com lending guide](https://eco.com/support/en/articles/14800882-best-defi-lending-protocols-2026-tvl-rates-risk) |
| Active loans (network-wide) | ~$4B | 2026-04/05 | [Phemex / DefiLlama summary](https://phemex.com/news/article/morpho-tvl-reaches-1178b-secures-second-place-in-defi-lending-80656) |
| Active loans on Base | >$1.18B (1,000% YoY growth) | 2026-01 | [Crypto Times](https://www.cryptotimes.io/2026/01/13/morpho-crosses-1-billion-in-active-loans-on-base-network/) |
| Annualised fees | ~$175M | 2026-04 | [Phemex / DefiLlama summary](https://phemex.com/news/article/morpho-tvl-reaches-1178b-secures-second-place-in-defi-lending-80656) |
| Coinbase BTC-backed loans (via Morpho) | $1B+ originated; $1.4B cbBTC collateral; $960M+ active loans | 2025-2026 | [The Block](https://www.theblock.co/post/373032/coinbase-tops-1-billion-in-bitcoin-backed-onchain-loans-via-morpho), [Morpho story](https://morpho.org/stories/coinbase/) |
| Codebase size (Morpho Blue core) | ~650 lines of Solidity | 2024 | [Morpho Blue launch blog](https://morpho.org/blog/morpho-blue-and-how-it-enables-our-vision-for-defi-lending/) |

### Design decisions

- **Singleton over factory.** One contract, many markets keyed by
  market id. Lower gas, unified flash-loan liquidity across all
  markets, simpler accounting.
- **Immutable code.** Removes governance attack surface; forces risk
  management to live above the protocol.
- **Permissionless market creation.** Any party can deploy a new
  market by selecting `(loanAsset, collateralAsset, oracle, IRM,
  LLTV)`. Governance controls only the IRM whitelist and the
  discrete LLTV grid.
- **Oracle agnostic per market.** Each market specifies its own
  oracle contract. Vault curators (and direct suppliers) carry
  oracle-vetting responsibility.
- **Adaptive Curve IRM.** Immutable, autonomous, targets 90%
  utilisation. No per-market governance retuning.
- **Permissionless liquidation.** Any actor may repay an unhealthy
  position in exchange for collateral plus a liquidation incentive.
- **Socialised bad debt within a single market.** If a liquidation
  cannot fully repay the debt, the residual is socialised across
  that market's lenders only; it cannot contaminate other markets.
- **Bundlers.** Smart contracts that bundle approve + wrap + supply
  + borrow into one transaction, inheriting from `BaseBundler`. See
  [morpho-blue-bundlers](https://github.com/morpho-org/morpho-blue-bundlers).
- **Gas profile.** Morpho's launch blog claims a ~70% reduction in
  gas consumption versus existing lending protocols (no specific
  Aave/Compound benchmark is given).

### MetaMorpho (curated vault layer)

MetaMorpho is the canonical curated-vault layer above Morpho Blue.
Each vault is an ERC-4626 contract accepting deposits in a single
loan asset and reallocating them across a curator-whitelisted basket
of Morpho Blue markets, subject to per-market supply caps and
timelocks.

| Aspect | V1 / V1.1 | V2 (2025-09-29) |
|--------|-----------|-----------------|
| Maximum markets per vault | 30 | Adapter-based, broader |
| Risk caps | Per-market absolute cap | ID-based caps (collateral asset, market, protocol) |
| Roles | Owner, Curator, Allocator, Guardian | Same, plus **configurable role segregation** for institutional separation-of-duties |
| Timelock | 24h minimum on risk-relevant changes | Same |
| Inflation attack mitigation | ERC-4626 virtual offset | Same |
| Redemption | Standard ERC-4626 | **In-kind redemption** via flash loan; users never locked in even when underlying markets are illiquid (a configurable exit penalty applies; the specific cap requires a docs citation, see follow-up) |

Source: [Morpho Vaults V2 announcement](https://morpho.org/blog/morpho-vaults-v2-a-new-standard-for-asset-curation/).

#### Guardian semantics

The Guardian deserves a closer look because the role's safety
guarantee depends on a subtlety that is easy to miss. The Guardian
cannot initiate parameter changes, only veto a subset of them, and
only inside the timelock window before they are accepted. Per the
Morpho Vaults docs ([Roles & Capabilities](https://docs.morpho.org/morpho-vaults/concepts/roles/)),
the Guardian can revoke:

- a pending **timelock decrease**, until the previous timelock ends
  and the new timelock is accepted;
- a pending **guardian** (i.e. a pending replacement of the
  incumbent), until the timelock ends and the new guardian is
  accepted by the Owner;
- each pending **market cap increase**, until the timelock ends and
  the new cap is accepted by the Owner or the Curator.

The Guardian **cannot** revoke a pending fee change. This is
material for threat modelling: fee adjustments are the one
governance lever a compromised owner can push through without
guardian interference.

The interesting property is the second bullet: the incumbent
guardian can revoke its own replacement. Combined with the timelock,
this means a single owner-key compromise does not let an attacker
race the window by also rotating the guardian; the incumbent stays
in office during the window and can block the malicious change.
The protection degrades to zero only if owner and guardian are
compromised simultaneously, which is a stronger threat model than
either single-key compromise.

##### Beyond default settings: making the guardian itself sticky

Two production deployments have published patterns that further
constrain who can rotate the guardian. They use **different
mechanisms** despite often being grouped together:

- **B.Protocol's LP-vote guardian.** A Gnosis Safe with an oSnap
  (UMA optimistic oracle) module is set as the Guardian. The Safe's
  Snapshot space uses the vault's own LP token as the single voting
  token, so vault depositors vote with their share balances. The
  Safe owner is set to a null address so no individual can rotate
  the guardian. The B.Protocol article notes the setup is **not
  fully trustless**: if UMA's optimistic oracle or the oSnap module
  malfunction, the Guardian becomes passive. See
  [B.Protocol: A Decentralised Guardian Setup](https://medium.com/b-protocol/a-decentralized-guardian-setup-how-metamorpho-lps-veto-risk-curator-actions-bf6653f6425a).
- **Steakhouse's Aragon-DAO guardian.** A wrapped governance token
  (`gsteakETH`) governs an Aragon DAO that holds the Guardian role.
  Critical Aragon permissions (`UPDATE_VOTING_SETTINGS`,
  `ROOT_PERMISSION_ID`, `UPGRADE_PLUGIN`) are revoked so no party
  can change the voting rules or rotate the guardian. See
  [Steakhouse: Trustless Guardian Upgrade](https://hackmd.io/Ahi_eTrOSm29PJEbvsG0FA).

Both achieve a "guardian cannot be rotated by a single party"
property; neither is fully trustless in the strict sense (each
depends on its underlying voting infrastructure remaining honest
and operational).

### Limitations and criticisms

- **Risk transfer to curators.** Depositors must trust curator
  competence and honesty. The
  [PAXG oracle misconfiguration](https://docs.morpho.org/security/)
  incident in 2024 ($230k loss) is an example of curator-side risk.
- **Oracle freedom is double-edged.** A market creator can deploy a
  market with an inaccurate oracle and attract deposits via yield
  boosting; direct suppliers carry that full risk. Vaults mitigate
  it by whitelisting markets.
- **Bug fixes require redeployment** and user migration.
- **Frontend dependency.** A 2025 frontend vulnerability (caught by
  white-hat disclosure, zero loss) shows protocol security is not
  equivalent to product security.
- **Curator concentration.** A handful of curators (Gauntlet,
  Steakhouse, Re7, MEV Capital, Block Analitica, Apostro) underwrite
  most Morpho TVL. Whether this is materially less centralised than
  DAO-controlled risk parameters in Aave is an open question.

## Security Record (2020-2026)

Cumulative reported losses across major lending-protocol incidents.
Where funds were returned, gross loss is shown with recovery in
brackets.

| Incident | Year | Gross loss USD | Recovered | Type |
|----------|------|----------------|-----------|------|
| bZx incidents | 2020 | ~$0.95M | None | Flash-loan + AMM oracle |
| Compound COMP bug | 2021 | $80M (in COMP rewards; no user funds) | Partial via voluntary returns | Governance upgrade bug |
| Cream AMP | 2021 | $25M | None | Reentrancy |
| Cream yUSD | 2021 | $130M | None | Share inflation |
| Solend whale crisis | 2022 | $0 (no exploit) | n/a | Governance process failure (failed SLND1 vote) |
| Inverse Finance (Apr) | 2022 | $15.6M | None | Oracle manipulation |
| Inverse Finance (Jun) | 2022 | $5.8M | None | Oracle manipulation (LP token) |
| Mango Markets exploit | 2022 | $110M | $67M | Oracle manipulation (perp underlying) |
| Iron Bank saga | 2022 | Tens of millions | Partial | Counterparty default (FTX collapse) |
| Euler V1 hack | 2023 | $197M | $197M (full return) | Flash-loan + missing solvency check |
| Hundred Finance | 2023 | $7M | None | Share inflation on Optimism |
| MarginFi team incident | 2024 | $0 contract loss; $155M+ outflows | n/a | Operational / governance |
| Radiant exploit (Jan) | 2024 | $4.5M | None | Rounding error / flash loan |
| Radiant exploit (Oct) | 2024 | $58M | None | Multisig key compromise (UNC4736) |
| Morpho PAXG oracle misconfig | 2024 | $230k | None | Market-level oracle misconfig |
| Morpho frontend vuln | 2025 | $0 (white hat) | n/a | Frontend bug |

### Patterns

- **Oracle attacks** are the single largest category by frequency
  ($300M+ across Mango, Inverse, Hundred, Cream, Morpho PAXG, and
  others).
- **Reentrancy and share inflation** are essentially solved for new
  protocols, but old Compound V2 forks continue to be exploited
  ($160M+ at Cream and Hundred).
- **Insolvency-check bugs**: Euler V1 ($197M, returned).
- **Multisig and governance compromises** are the emerging
  high-impact attack vector for any protocol with an upgradeable
  proxy (Radiant Oct 2024, $58M).
- **Frontend / SDK / off-chain pipeline vulnerabilities** are an
  under-addressed surface (Morpho 2025).
- **Operational / team incidents** can cost more than many contract
  exploits without involving any code defect (MarginFi 2024,
  ~$155M+ in user outflows).

### Implication

Protocols that minimise governance surface (immutable core),
choose oracles per market with curator vetting (curated vaults),
and isolate markets (one bad market does not contaminate another)
materially reduce the most common loss vectors observed since 2020.

## Open Questions in the Ecosystem

These are unresolved design questions across the surveyed protocols
as of mid-2026. They surface in the
[RFP-008](../RFPs/RFP-008-lending-borrowing-protocol.md) and
[RFP-012](../RFPs/RFP-012-curated-lending-vaults.md) requirements
discussions:

- **Curator concentration.** Is curator-driven risk management
  functionally as centralised as DAO-controlled risk parameters,
  given that ~6 firms underwrite most Morpho TVL?
- **Bitcoin lending without wraps.** Can Babylon-style script-
  enforced collateral replace LST-bridge models for BTC-backed
  loans, eliminating the consortium custody point?
- **RWA scale.** Aave V4's hub-and-spoke and Sky's allocator vaults
  signal a coming wave of RWA-backed DeFi credit. The legal-default
  machinery to handle off-chain breaches in a DeFi-native way
  remains immature.
- **Frontend / SDK security.** The Morpho 2025 incident reinforces
  that protocol security is not equivalent to product security.
  Bundler-level invariant checks, transaction simulation, and
  in-app proof-of-action UIs are under-developed across the
  ecosystem.
- **Solana isolation vs rehypothecation.** Whether
  "isolated-with-some-rehypothecation" is a viable middle ground
  (Jupiter Lend) or a marketing fudge (per Kamino's public
  pushback) remains a live design debate.
- **Long-tail asset listing.** Aave's slow listing process is a
  source of friction; permissionless models address this but
  require curator vetting. The right balance for institutional
  users is still being discovered.

## Sources

Primary protocol documentation:

- Aave: [Aave V3 Overview](https://aave.com/docs/aave-v3/overview),
  [DefiLlama: Aave](https://defillama.com/protocol/aave)
- Morpho:
  [Morpho Blue Whitepaper](https://resources.cryptocompare.com/asset-management/17952/1732199021661.pdf),
  [Morpho Docs](https://docs.morpho.org),
  [DefiLlama: Morpho Blue](https://defillama.com/protocol/morpho-blue)
- Compound:
  [Compound III Docs](https://docs.compound.finance/),
  [DefiLlama: Compound V3](https://defillama.com/protocol/compound-v3)
- Spark: [spark.fi](https://spark.fi),
  [Alea Research: Spark](https://alearesearch.io/perspectives/spark/)
- Kamino:
  [Kamino Docs](https://docs.kamino.finance/),
  [Crypto Adventure: Kamino review 2026](https://cryptoadventure.com/kamino-review-2026-solana-lending-vaults-leverage-and-liquidation-risk/)
- Euler:
  [Euler Finance](https://www.euler.finance),
  [eco.com: Euler V2](https://eco.com/support/en/articles/14800904-euler-v2-modular-lending-vault-design)
- Silo: [Silo Docs](https://docs.silo.finance/docs/users/intro/)
- Fluid: [Fluid Docs](https://docs.fluid.io)
- Maple:
  [Maple Docs](https://docs.maple.finance/),
  [Vaasblock: Maple syrupUSD 2026](https://www.vaasblock.com/research/maple-finance-syrup-token-risks-onchain-credit-2026/)
- Babylon:
  [Babylon Docs](https://docs.babylonchain.io),
  [PANews BTCFi report](https://www.panewslab.com/en/articles/0mmtz1k6)
- Lombard:
  [lombard.finance](https://lombard.finance),
  [Starknet: LBTC on Starknet](https://www.starknet.io/blog/lombard-brings-bitcoin-to-starknet-through-lbtc-integration/)
- Liquidium:
  [liquidium.fi](https://liquidium.fi),
  [DL News: Maestro / Liquidium](https://www.dlnews.com/research/external/maestro-advances-native-ordinals-and-runes-indexer-on-icp-to-power-bitcoin-defi/)

Ecosystem and metric coverage:

- [eco.com: Best DeFi Lending Protocols 2026](https://eco.com/support/en/articles/14800882-best-defi-lending-protocols-2026-tvl-rates-risk)
- [Phemex / DefiLlama: Morpho TVL $11.78B](https://phemex.com/news/article/morpho-tvl-reaches-1178b-secures-second-place-in-defi-lending-80656)
- [The Block: Coinbase $1B BTC-backed loans](https://www.theblock.co/post/373032/coinbase-tops-1-billion-in-bitcoin-backed-onchain-loans-via-morpho)
- [The Block: Jupiter Lend rehypothecation backlash](https://www.theblock.co/post/381602/jupiter-exec-acknowledges-zero-contagion-claim-was-not-100-correct-after-backlash-over-vault-design)
- [Crypto Times: Morpho $1B on Base](https://www.cryptotimes.io/2026/01/13/morpho-crosses-1-billion-in-active-loans-on-base-network/)
- [Coindesk: Coinbase USDC lending with Morpho + Steakhouse](https://www.coindesk.com/business/2025/09/18/coinbase-adds-usdc-lending-with-morpho-and-steakhouse-financial)
- [Volity: Compound 2026 guide](https://volity.io/crypto/compound/)
- [Mixbytes: Aave V3 architecture](https://mixbytes.io/blog/modern-defi-lending-protocols-how-its-made-aave-v3)
- [Cyfrin: Aave V3 risk management](https://www.cyfrin.io/blog/aave-v3-improved-lending-liquidity-and-risk-management)

Design patterns:

- [Morpho: Adaptive Curve IRM blog](https://morpho.org/blog/introducing-the-adaptivecurveirm-efficient-and-autonomous/)
- [Morpho: Vaults V2 announcement](https://morpho.org/blog/morpho-vaults-v2-a-new-standard-for-asset-curation/)
- [Morpho Labs on X: 600 LOC + audits](https://x.com/MorphoLabs/status/1768264582634836055)

This appendix synthesises material from the
[research-lending](https://github.com/marclawclaw/research-lending)
Obsidian vault. Per-protocol notes, security incident
post-mortems, and full source lists live there.
