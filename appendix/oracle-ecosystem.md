# Appendix: Oracle Ecosystem

This appendix surveys oracle protocols, TWAP mechanics, manipulation
vectors, and external oracle models relevant to
[RFP-019](../RFPs/RFP-019-twap-oracle.md). It provides the technical
and market context for the two-tier oracle architecture proposed for
LEZ.

## Oracles Surveyed

Protocols are ordered by Total Value Secured (TVS), largest first.
This order is maintained throughout the document.

| Protocol | TVS | Chains | Model | Feed Count | Key Feature |
|----------|-----|--------|-------|------------|-------------|
| Chainlink | $66B-$75B | 27 push / 60+ via CCIP | Push (OCR/DON) | 1,000+ | Decentralised Oracle Network with VWAP from premium data aggregators |
| Chronicle | $10.2B+ | 13 | Push | Limited | MakerDAO-native; concentrated TVS from Sky's $10B+ TVL |
| Pyth | $8.6B+ | 50+ via Wormhole | Pull (Wormhole) | 2,800+ | First-party data from 120+ institutional publishers; confidence intervals |
| RedStone | $10B+ | 50+ push / 120+ pull | Pull (calldata) | 1,000+ | No bridge dependency; modular push+pull; fastest-growing oracle |
| Switchboard | $3B+ | 9 | Pull (TEE) | Permissionless | TEE (SGX/SEV) security; permissionless custom feed creation |
| Supra | $650M+ | 45 | Push+Pull | N/A | Newer entrant; DORA (Distributed Oracle Agreement) consensus |

## Scale and Traction

Total Value Secured measures the aggregate DeFi TVL in protocols that
depend on a given oracle. An oracle with higher TVS has withstood
more economic scrutiny, though TVS is a trailing indicator: new chains
start at $0 regardless of oracle choice.

### Market share

Chainlink dominates with approximately 68% of global oracle TVS and
over 80% on Ethereum specifically [1]. Chronicle's high TVS
(approximately 17%) is concentrated in a single protocol
(MakerDAO/Sky) and does not reflect general-purpose adoption [2].
Pyth leads on data quality through first-party institutional
publishers; per Pyth's own April 2026 positioning, the network reaches
"100+ blockchains" via Wormhole, with cross-chain support
established on roughly 50+ chains in production. RedStone has the
fastest growth trajectory, driven by explicit support for L2s,
appchains, and rollups; RedStone reports no oracle-induced mispricing
on its flagship integrations (Ethena, Gearbox) through early 2026
[2][3][17].

### Per-protocol adoption

**Chainlink.** Deployed on 27 chains with push model and 60+ public
and private blockchains via CCIP. Network integration requirements
include high-availability RPC providers, valid SSL, Ethereum
JSON-RPC compatibility, and 30-day historical RPC performance
metrics, making it inaccessible for new chains at launch [4]. Uses
Off-Chain Reporting (OCR) where nodes aggregate prices from premium
data vendors (Kaiko, CoinMetrics) and submit a single signed
transaction per round. Update triggers are deviation threshold (e.g.
0.5% price change) or heartbeat (e.g. 1 hour maximum staleness).

**Pyth.** Originated on Solana and now cross-chain via Wormhole.
Aggregates first-party data from 120+ institutional publishers
(April 2026), including Jane Street, Coinbase, and Binance, across
2,800+ price feeds (December 2025). Architecture: publishers post
prices on Pythnet (a Solana appchain) every 400ms; Wormhole guardians
sign a Merkle root; the Hermes off-chain cache stores latest proofs;
users fetch and submit proofs in their transactions. Every price
includes a confidence interval, enabling protocols to reject
high-uncertainty updates. Known incidents include the September 2021
BTC/USD mispricing, where the feed reported approximately $5,402
against an actual price near $43,500 (an approximately 87% drop)
[37], and the March 2025 Morpho cbETH wrongful liquidation on Base,
where a roughly 7-minute cbETH/ETH staleness window distorted the
collateral price and liquidated a user for approximately $33,000 on
a Re7 Labs vault [38].

**RedStone.** Modular oracle with both push and pull delivery. Pull
model attaches signed data to EVM calldata; the on-chain contract
verifies node signatures without requiring a bridge or dedicated
relay infrastructure [3]. Fastest-growing oracle in 2024 to 2025,
with deployments on Monad, Hyperliquid (HyperStone), and 120+ pull
chains. Zero reported mispricing incidents as of early 2026 [3][17].
Expanding into RWA feeds (BlackRock BUIDL, VanEck VBILL) and risk
ratings via Credora acquisition.

**Switchboard.** Permissionless feed creation via TEE (SGX/SEV)
oracle nodes [13]. Any developer can create custom feeds for assets
not covered by Pyth or RedStone. Can aggregate from multiple upstream
oracles (Pyth + Chainlink + custom APIs) in a single feed. The core
protocol contract is a single-deployment artefact per chain
(deployed by the Switchboard team), but feed creation on top of it
is permissionless; TEE hardware requirements limit the operator pool.

**DIA Lumina.** Fully permissionless: both data sourcing (Feeder
nodes) and feed deployment (Aggregator contracts) require no team
permission [7]. ZK proof verification aligns with privacy-focused
chains. Smaller feed catalogue (200+) and newer ZK architecture
(Lumina V2) make it less battle-tested than Pyth or RedStone.

## TWAP Mechanics

### Uniswap v2 accumulator

Each Uniswap v2 pool stores `price0CumulativeLast` and
`price1CumulativeLast` variables. These accumulators are updated at
the beginning of each block (before any same-block trades execute)
using the price set by the last trade of the previous block [14]. The
formula: `cumulativePrice += price * timeElapsed`. To compute a TWAP,
an external contract reads the accumulator at two timestamps (T1 and
T2) and divides:

```
TWAP = (accumulator[T2] - accumulator[T1]) / (T2 - T1)
```

This yields an arithmetic mean price. No off-chain components are
required.

### Uniswap v3 accumulator

Uniswap v3 replaced the raw price accumulator with a tick-based
accumulator (`tickCumulative`), which stores the running sum of
`currentTick * secondsElapsed` [9]. The TWAP is:

```
averageTick = (tickCumulative[T2] - tickCumulative[T1]) / (T2 - T1)
price = 1.0001^averageTick
```

This yields a geometric mean price, which is more appropriate for
multiplicative price processes. The observation buffer is a circular
array of up to 65,535 slots, expandable via
`increaseObservationCardinalityNext()` at a one-time gas cost [9].

### Geometric vs arithmetic mean

The arithmetic mean (v2) is sensitive to outliers: an attacker who
moves the price up by 10x in one block and back by 10x in the next
leaves a net upward bias in the arithmetic mean. The geometric mean
(v3) is invariant to such symmetric multiplicative manipulation,
because `log(10x) + log(1/10x) = 0` [9]. For this reason, the v3
approach is strictly preferred for price oracle use.

### Example computation (v3)

```
tickCumulatives = [70,000, 1,070,000] over 10 seconds
averageTick = (1,070,000 - 70,000) / 10 = 100,000
price = 1.0001^100,000 = 22,015.5 USDC/WETH
```

Minor imprecision arises because ticks are integers and fractional
ticks are truncated, but this is negligible for most use cases [9].

## TWAP Manipulation Vectors

### Flash loan attacks (within-block)

An attacker borrows massive capital via a flash loan, moves the pool
price heavily, and profits from a protocol that reads the manipulated
price in the same or next block. Under Uniswap v2/v3, the accumulator
is updated at block start (before same-block trades), so within-block
manipulation does not affect the current block's accumulator sample.
However, the attacker can manipulate price at the end of block N,
which contaminates the accumulator sample in block N+1 [6][10].

### PoS multi-block validator attacks

Under Proof of Stake, validators know one epoch ahead (32 blocks on
Ethereum, approximately 6.4 minutes) whether they control consecutive
blocks. A validator controlling two consecutive blocks can move the
price in block N and reverse it in block N+1, at a cost approximately
equal to the round-trip swap fees and price impact, with no back-run
competition [6]. This contaminates one accumulator data point per
attack. On high-liquidity pools (e.g.
USDC/WETH 5bps on Ethereum), the attack is economically infeasible;
on low-liquidity pools, it is trivially cheap.

### Low-liquidity vulnerability

Manipulation cost scales approximately linearly with pool liquidity
depth [6][10]. On the deep USDC/WETH 5bps pool, adding $1M of
wide-range liquidity raises the marginal cost of a two-block oracle
attack by approximately $360B above the already-large baseline,
demonstrating extreme sensitivity [6]. On a new chain like LEZ with
$1M pools, the same attack that costs trillions on Ethereum mainnet
costs only thousands.

### Short observation window attacks

An attacker sustains a manipulated price across multiple blocks for
the duration of a short TWAP window (e.g. 5 minutes). This is more
expensive than single-block manipulation but feasible on low-liquidity
pools [10]. The cost scales with both window length and pool depth.

### Historical losses

According to the Rekt Database (cited in the Ormer paper), 36 flash
loan oracle attacks alone caused over $418M in cumulative losses [5].
Oracle manipulation is the primary attack vector in DeFi exploits.

## Window Selection Tradeoffs

### The core tradeoff

Short TWAP windows (e.g. 5 minutes) provide fresh prices but are
cheap to manipulate. Long windows (e.g. 24 hours) are expensive to
manipulate but lag the market severely during genuine volatility.
Unlike external oracle networks, AMM TWAP cannot simultaneously
optimise both security and freshness [10][11].

### Production standards

The majority of TWAP oracles in production DeFi use windows between
30 minutes and 12 hours [11].

| Use case | Typical window | Rationale |
|----------|---------------|-----------|
| Lending collateral valuation | 30 min to 2 h | Resist short-term manipulation; liquidation timing tolerable |
| DEX AMM internal pricing | 5 to 30 min | Requires responsiveness for arbitrage |
| Governance and voting | 24 h to 7 days | Resist flash attacks on governance weight |

### Cardinality math

At 12-second blocks, the maximum observation cardinality of 65,535
provides approximately 218 hours (roughly 9 days) of lookback [9].
Protocols trading off storage cost vs lookback depth should expand
cardinality to at least `(desired_window_seconds / block_time) + buffer`.
For a 2-hour window at 12s blocks, minimum cardinality is 600 + buffer.

### Manipulation cost scaling

Moving the price by 5% on a 1-hour TWAP requires sustaining that 5%
deviation every block for 1 hour, at a cost approximately equal to
the arbitrage losses and fees incurred per block, multiplied by the
number of blocks [10]. Doubling pool liquidity approximately doubles
the cost; doubling the window length approximately doubles the cost.

## External Oracle Models

### Push vs pull

Push oracles (Chainlink) submit price updates to an on-chain contract
on a heartbeat (e.g. every hour) or deviation trigger (e.g. 0.5%
price change). Gas cost per update ranges from 300K to 2.4M gas
depending on the aggregation method [4]. Pull oracles (Pyth, RedStone)
sign data off-chain; the consumer fetches the signed data and submits
it as part of their own transaction, paying the verification gas cost
(approximately 50K to 100K gas per update) [3][5].

For new chains, pull is strongly preferred: no dedicated per-chain
node operators, no ongoing gas subsidies, and immediate availability
once the verification contract is deployed.

### DON vs single-source

Chainlink's Decentralised Oracle Network (DON) uses Off-Chain
Reporting (OCR) where multiple independent nodes each fetch prices
from premium data aggregators, communicate via P2P, elect a leader
who produces a signed report containing all observations, and submit
the median on-chain [12]. This is technically VWAP (volume-weighted
from multi-exchange aggregation), not a strict TWAP from a single
AMM [12].

Single-source oracles (a single AMM TWAP) offer maximum
trustlessness but depend entirely on that source's liquidity and
availability. DON-style oracles trade some trustlessness (off-chain
node operators) for dramatically better manipulation resistance and
market coverage.

### Comparison table

| Dimension | AMM TWAP | Pyth | RedStone | Chainlink DON |
|-----------|---------|------|----------|---------------|
| Trust model | Trustless (on-chain) | Semi-trusted (publishers + Wormhole) | Semi-trusted (node signatures) | Semi-trusted (node operators + staking) |
| Data source | Single DEX pool | 70+ first-party publishers | CEX + DEX + aggregators | Premium aggregators (Kaiko, CoinMetrics) |
| Market coverage | On-chain pairs only | 1,500+ feeds (crypto, FX, equities) | 1,000+ feeds | 1,000+ feeds |
| New-chain deployability | Requires AMM with liquidity | Requires Wormhole | No bridge needed | Requires 3+ RPC providers, node operators |
| Manipulation resistance | Scales with pool depth | Independent of on-chain liquidity | Independent of on-chain liquidity | Independent of on-chain liquidity |
| Gas per query | Very low (read accumulator) | 50K to 100K (VAA verification) | 50K to 100K (signature verification) | N/A (push: consumer reads storage) |
| Confidence interval | No | Yes | No | No |
| Real-world assets | No | Yes | Yes (RWA feeds) | Yes |

## Signature Verification Schemes

The cost of verifying oracle signatures on-chain dictates whether
LEZ can host an oracle adaptor at all. Since LEZ inherits the
Solana Virtual Machine architecture, it has native precompiles for
secp256k1 ECDSA and ed25519 EdDSA, but no built-in support for
threshold or aggregate schemes (BLS, Schnorr multisig, t-Schnorr).
This section verifies, against primary sources, what schemes Pyth
(via Wormhole) and RedStone use, and what they cost.

### Wormhole VAA verification (Pyth dependency)

A Verifiable Action Approval (VAA) is the canonical Wormhole
attestation. The guardian set is a fixed roster of 19 nodes; a
super-majority of 13 signatures is required for a VAA to be
considered valid [19][20]. Each guardian signs the
double-keccak256 hash of the VAA body (single keccak256 on Solana,
because the Solana secp256k1 program hashes the message itself)
[19]. The signature scheme is plain m-of-n ECDSA over secp256k1,
with no aggregation: the VAA carries 13 independent (v, r, s)
tuples plus a guardian index per signature [19][21]. Wormhole's
own protocol documentation explains the choice: BLS aggregation
would cost roughly 130K gas to verify on Ethereum, whereas plain
ECDSA via the ecrecover precompile costs about 5K gas per
signature, so the simpler scheme wins on cost despite the linear
verification work [21][22].

On EVM chains, the core bridge contract calls verifyVM, which
invokes verifySignatures and loops over the 13 signatures one by
one, calling ecrecover for each [21][22]. The ecrecover
precompile has a fixed base cost of 3,000 gas [23]; combined with
calldata costs, the documented per-signature cost is
approximately 5K + 5K (recovery + calldata), giving roughly 130K
gas for the signature step alone before Merkle proof verification
and storage writes [22]. On Solana, Wormhole calls the native
secp256k1 precompile via verify_signatures, but Solana's per-call
compute-unit budget forces the verification to be split across
multiple instructions, each verifying six or seven signatures at
a time [22][24]. The Solana secp256k1 precompile costs
approximately 6,690 compute units per signature verified [24][25].
For Pyth specifically, the Perseus upgrade amortises the VAA
verification step across multiple price updates: a single set of
Wormhole signatures is verified once per transaction regardless
of how many feeds are updated, yielding a 50 to 80 per cent gas
reduction when updating five feeds at once [26].

Alternative verification paths exist but are not in production
for Pyth. Wormhole has shipped a Boundless ZK verifier (RISC Zero
Groth16 proofs of Ethereum consensus) deployed on Ethereum, Base,
Optimism, Arbitrum, Linea, and Avalanche, with a two-of-two
policy that requires both a valid guardian signature set and a
valid ZK proof before a transfer executes [27]. The
wormhole-foundation/example-zk-light-clients repository contains
ZK light client templates for various source chains [27]. None of
these supersede the 13-of-19 ECDSA path for Pyth price-feed
delivery; they augment it.

### RedStone per-chain signatures

RedStone signs every data package with one scheme everywhere it
operates: ECDSA over secp256k1 with a keccak256 message hash,
matching Ethereum's signing convention [28][29]. The reference
implementation lives in the redstone-finance/rust-sdk crate; its
Cargo.toml depends on either the secp256k1 crate (with the
recovery feature) or the k256 ECDSA crate, plus sha3 for keccak
[29]. The crypto module exposes recover_public_key and
recover_address functions that validate signature malleability
against the secp256k1 curve order and accept Ethereum-style
recovery bytes (0, 1, 27, 28) [29]. The on-chain EVM consumer
contract calls SignatureLib.recoverSignerAddress on
keccak256(signedMessage), then enforces a per-feed unique-signer
threshold via getUniqueSignersThreshold(); RedStone's
documentation recommends at least three unique signers as a
balance between security and gas cost [28][30].

The signing scheme does not change per chain; only the
verification primitive does. Each chain-specific connector
recovers the same secp256k1 ECDSA signatures using whatever
host-chain primitive is available:

| Chain | Verification primitive | Source |
|-------|-----------------------|--------|
| EVM | ECRECOVER precompile (secp256k1, keccak256) | [28][30] |
| Solana | secp256k1_recover syscall / secp256k1 program | [25][29] |
| Sui | sui::ecdsa_k1::secp256k1_ecrecover Move builtin | [31] |
| Stellar (Soroban) | recover_key_ecdsa_secp256k1 host function | [32][33] |
| Fuel | Sway contract using secp256k1 recovery | [34] |
| Radix (Scrypto) | Rust SDK with secp256k1 / k256 crate | [29][35] |
| Casper | Rust SDK with secp256k1 / k256 crate | [29] |

The reviewer's claim that RedStone uses ed25519 on Stellar
appears to derive from the DeployingFeed.md note that "the
private key here can be any 256-bit hex string, because stellar
uses the Ed25519-curve" [36]. That sentence refers to the
deployer's Stellar account key (which Stellar requires to be
ed25519 for its native account model), not to the curve used to
sign or verify RedStone data packages. The Stellar connector's
audit by Veridise describes the Soroban contract verifying
ECDSA signature parameters from RedStone's payload, not ed25519
signatures [33]. RedStone signs once with secp256k1 / keccak256
and verifies the same signatures everywhere, including on
Stellar via Soroban's recover_key_ecdsa_secp256k1 host function
(documented CPU cost: 2.3 million instructions per recovery)
[32].

Per-chain verification cost (single signature):

| Chain | Cost per ECDSA recovery | Source |
|-------|------------------------|--------|
| Ethereum / EVM | 3,000 gas (precompile) plus calldata | [23] |
| Solana | approximately 6,690 compute units | [24][25] |
| Stellar (Soroban) | 2.3M CPU instructions | [32] |

For an M-of-N RedStone payload with the recommended threshold
of three unique signers [30], total verification cost on EVM is
approximately 9,000 gas for the precompile calls plus calldata
and signer-bitmap accounting; the documented end-to-end gas cost
of a RedStone update on EVM falls in the 50K to 100K range [3]
(consistent with the table in External Oracle Models above).

### Implications for LEZ

LEZ inherits Solana's architecture, which provides native
precompile programs for both secp256k1 ECDSA (with keccak256 or
sha256 hashing) and ed25519 EdDSA [25]. Both Pyth (via Wormhole
VAAs) and RedStone (per-chain connectors) can therefore be
ported to LEZ without new opcodes or runtime changes: Pyth needs
a Wormhole core-bridge port that calls the secp256k1 program in
batches of seven (mirroring the existing Solana implementation)
[22], and RedStone needs only its existing solana-connector with
the secp256k1_recover syscall [29]. RedStone is the cheaper and
simpler day-one option because it requires no bridge: a single
verification contract recovers three to five secp256k1
signatures from calldata, costing approximately 20K to 35K
compute units total. Pyth depends on the full 13-of-19 VAA
verification, costing approximately 87K compute units for the
signature step alone (13 signatures multiplied by 6,690 CU)
plus Merkle proof verification, but amortises across many feeds
per transaction after the Perseus upgrade [26]. No threshold or
aggregate scheme (BLS, Schnorr) is required by either oracle, so
LEZ does not need to implement new cryptographic precompiles to
host both adaptors.

## Production Oracle Architectures

Major lending and borrowing protocols have converged on multi-source
oracle designs with fallback mechanisms. These production patterns
inform the requirements and design rationale of RFP-019.

### Aave V3

Aave V3's `AaveOracle` contract uses Chainlink aggregators as the
primary price source via `getAssetPrice()`. Each asset is mapped to a
Chainlink feed through `setAssetSources()`. If the Chainlink feed
returns a price <= 0, the call is forwarded to a configurable fallback
oracle via `getFallbackOracle()` [15]. On Layer 2 deployments, the
`PriceOracleSentinel` contract monitors sequencer uptime: if the L2
sequencer goes down, borrowing is disabled and liquidations are paused
for a configurable grace period (`setGracePeriod()`), giving users
time to restore position health after an outage [15]. Aave does not
use TWAP as a primary or secondary price source; it relies entirely
on Chainlink feeds with governance-managed fallback.

### Compound V2 and V3

Compound V2's `UniswapAnchoredView` contract implemented a two-source
design: Coinbase as the primary reporter, anchored against a Uniswap
V2 TWAP. If the Coinbase price diverged beyond 20% of the TWAP
anchor, the price was rejected and the system retained the last valid
price [16]. This is the closest production precedent to RFP-019's
circuit breaker design. On 26 November 2020, a DAI price spike to
$1.30 on Coinbase triggered approximately $89M in liquidations;
the TWAP anchor limited the damage by rejecting the most extreme
prices, but the 20% tolerance was too wide to prevent all
mispricing [16]. Compound V3 (Comet) dropped the TWAP anchor entirely
and uses Chainlink feeds directly via `AggregatorV3Interface`, with
only zero-value validation; some markets use derived-asset price
wrappers (e.g. wstETH/ETH multiplied by ETH/USD) but these are
conversion adaptors, not bounds-checking circuit breakers [16].

### MakerDAO / Sky

MakerDAO's oracle pipeline has two layers: the Median contract
(aggregates prices from multiple whitelisted feed providers) and
the Oracle Security Module (OSM), which imposes a mandatory one-hour
delay before new prices take effect in the system [15a]. The delay
is a circuit breaker by design: it gives governance one hour to
detect and respond to oracle manipulation before the system acts on
a compromised price. Emergency responses include calling `stop()`
to freeze the OSM or triggering Emergency Shutdown. MakerDAO does
not use Uniswap TWAP. Chronicle Protocol evolved from MakerDAO's
internal oracle infrastructure, using Schnorr signature aggregation
to consolidate validator signatures into a single "super signature,"
achieving constant-time verification at approximately 52K gas
regardless of the number of validators [15b].

### Liquity V2

Liquity V2 explicitly rejected Uniswap V3 TWAP as a primary or
fallback source due to liquidity migration risk (uncertainty about
whether v3 liquidity would persist after the Uniswap v4 launch)
[18]. Instead, Liquity V2 uses Chainlink as the primary oracle (composing multiple
Chainlink feeds for LST collateral, e.g. ETH/USD combined with an
LST market-rate feed) with a simple fallback: if the primary feed
is frozen for more than 12 hours or returns bad data, the system
falls back to a secondary oracle (once, with no cascading). Liquity V2 explicitly rejects
complex circuit breaker designs (e.g. Gyroscope's pause-on-divergence
model), reasoning that pausing operations requires human intervention
to set a new oracle and unpause, which conflicts with their
immutability goals [18]. The design prioritises automation: simple
trigger conditions, single fallback, no manual intervention required.

### Common patterns

| Protocol | Primary | Secondary / fallback | Circuit breaker | TWAP role |
|----------|---------|---------------------|-----------------|-----------|
| Aave V3 | Chainlink | Governance-set fallback | PriceOracleSentinel (L2 sequencer) | None |
| Compound V2 | Coinbase reporter | Last valid price | 20% TWAP anchor tolerance | Anchor / sanity check |
| Compound V3 | Chainlink | None (governance pauses) | Zero-value check only | None |
| MakerDAO | Median (multi-feed) | Emergency Shutdown | 1-hour OSM delay | None |
| Liquity V2 | Chainlink | Simple fallback (one hop) | None (by design) | Rejected |

Two patterns emerge. First, most major lending protocols use at
least two tiers of price validation (Aave, MakerDAO, Liquity V2,
Compound V2): cross-check, delay, or fallback. Compound V3 is the
notable exception, relying on Chainlink with only zero-value
validation and governance-managed pause as its safety net. Second,
TWAP serves as an anchor or sanity check in the protocols that use
it (Compound V2), never as the sole price source for lending. These
patterns motivate RFP-019's two-tier architecture (on-chain TWAP +
external feeds) and circuit breaker design as a defence-in-depth
choice rather than a universal industry default.

## Privacy-Asset Feed Availability

LEZ's privacy focus likely makes XMR/USD and ZEC/USD first-class
pricing requirements. On-chain TWAP cannot supply these prices: no
wrapped XMR or ZEC token has sufficient DEX liquidity (low five to
six figures across all surveyed pools on Ethereum and Solana) for a
manipulation-resistant TWAP, so an off-chain feed is the only viable
path on day one. Coverage across the surveyed oracles is summarised
below.

| Oracle | XMR/USD | ZEC/USD | Self-serve on LEZ? |
|--------|---------|---------|--------------------|
| Chainlink (push) | Active on Optimism (1,200s heartbeat, 0.2% deviation) and Polygon (24h, 1%) [39] | Active on Ethereum (24h, 2%) and Polygon (24h, 1%) [40] | No: permissioned onboarding |
| Chainlink Data Streams | 43+ chains, subscription-gated [39] | 35+ chains, subscription-gated [40] | No: paid product |
| Pyth (pull) | `crypto-xmr-usd`, approximately 80+ publishers across two clusters [41] | `crypto-zec-usd`, 28 publishers [41] | Yes, once Wormhole endpoint and the Pyth receiver are deployed |
| RedStone (pull) | Listed; data feed ID `XMR` [42] | Listed; data feed ID `ZEC` [42] | Yes: RedStone publishes a Solana / SVM connector built on the native secp256k1 precompile (the same scheme as its EVM and other chain connectors); no bridge dependency, no per-chain RedStone team engagement |
| DIA / Lumina | Production-ready (MAIR aggregation, 120s, announced January 2026) [43] | Available (MAIR, 120s) [43] | Yes via Lumina; bespoke deployment per chain |
| Supra | XMR_USDT, 195 sources (Standard tier) [44] | ZEC_USDT, 60 sources (Premium tier) [44] | No: requires Supra team engagement |
| Chronicle | Not in public feed catalogue | Not in public feed catalogue | N/A |
| API3 | Not found | Not found | N/A |
| Switchboard | Not found | Not found | Permissionless feed creation supported but no XMR or ZEC feed currently exists |

## LEZ Bootstrap Strategy

### Phase 1: Genesis (no TVL)

Deploy RedStone pull oracle contract (no bridge dependency) and, if
Wormhole is available, the Pyth EVM receiver contract. This provides
400 to 1,500+ price feeds immediately with zero per-chain oracle node
infrastructure [3][5]. RedStone is the practical day-one oracle
because it requires no cross-chain bridge; Pyth is conditional on
Wormhole integration [8].

### Phase 2: Early growth (first DEX, some TVL)

Once RFP-004 (DEX) is live, deploy the on-chain TWAP oracle as a
supplementary data source. At this stage, TWAP should be used only
as a sanity check (circuit breaker comparison against external feeds),
not as a primary price source, because pool liquidity will be
insufficient for manipulation resistance [10][11].

Engage Switchboard for core EVM contract deployment (TEE security,
permissionless custom feeds). Consider DIA Lumina as a permissionless
fallback for assets not covered by Pyth or RedStone [7].

### Phase 3: Maturity (significant TVL)

As DEX pools grow beyond $50M to $100M in depth, on-chain TWAP
becomes a viable secondary price source for lending and liquidation
use cases. At this stage, multi-source aggregation (median of TWAP +
Pyth + RedStone) provides the highest reliability [6].

Engage Chainlink for official DON deployment once LEZ meets
infrastructure requirements (3+ independent RPC providers, sufficient
TVL, dedicated node operators). Treat Chainlink as a 12 to 24 month
post-launch milestone [4][8].

### Graduation path

The TWAP tier's role evolves with liquidity:

| Pool depth | TWAP role | Recommended window |
|------------|-----------|-------------------|
| < $1M | Not usable (manipulation trivially cheap) | N/A |
| $1M to $10M | Sanity check only (circuit breaker) | 2 to 12 h |
| $10M to $50M | Secondary source (median with external) | 30 min to 2 h |
| > $50M | Co-primary source alongside external feeds | 30 min to 1 h |

## References

1. DefiLlama, "Oracles" dashboard, accessed Q1 2026.
   https://defillama.com/oracles
2. Mitosis University, "Which Oracle Powers What," 2025.
   https://university.mitosis.org/chainlink-pyth-redstone-chronicle-supra-switchboard-which-oracle-powers-what/
3. RedStone, "Blockchain Oracles Comparison: Chainlink vs Pyth vs
   RedStone 2025," Jan 2025.
   https://blog.redstone.finance/2025/01/16/blockchain-oracles-comparison-chainlink-vs-pyth-vs-redstone-2025/
4. Chainlink, "Network Integration Requirements."
   https://docs.chain.link/resources/network-integration
5. Bai et al., "Ormer: A Manipulation-Resistant and Gas-Efficient
   Blockchain Oracle Scheme," arXiv:2410.07893v2, Oct 2024.
   https://arxiv.org/html/2410.07893v2
6. ChainSecurity, "Oracle Manipulation After The Merge," 2022.
   https://chainsecurity.com/oracle-manipulation-after-merge/
7. DIA, "Lumina" documentation, accessed 2026.
   https://www.diadata.org/lumina/
8. Pyth Network, "Cross-Chain Delivery" documentation.
   https://docs.pyth.network/price-feeds/core/how-pyth-works/cross-chain
9. Uniswap, "Oracle" (v3 protocol concepts).
   https://docs.uniswap.org/concepts/protocol/oracle
10. Uniswap Labs, "Uniswap v3 Oracles" (PoS analysis), 2022.
    https://blog.uniswap.org/uniswap-v3-oracles
11. SmartContent, "TWAP Oracles vs Chainlink Price Feeds: A
    Comparative Analysis."
    https://smartcontentpublication.medium.com/twap-oracles-vs-chainlink-price-feeds-a-comparative-analysis-8155a3483cbd
12. Chainlink, "Off-Chain Reporting" documentation.
    https://docs.chain.link/architecture-overview/off-chain-reporting
13. Switchboard, "Protocol" documentation.
    https://docs.switchboard.xyz/how-it-works/switchboard-protocol
14. Uniswap, "Oracles" (v2 protocol concepts).
    https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles
15. Aave, "Oracles" (v3 smart contracts documentation).
    https://aave.com/docs/aave-v3/smart-contracts/oracles
15a. MakerDAO, "Oracle Security Module (OSM)" documentation.
     https://docs.makerdao.com/smart-contract-modules/oracle-module/oracle-security-module-osm-detailed-documentation
15b. Chronicle Labs, "Understanding Chronicle" documentation.
     https://docs.chroniclelabs.org/understandingChronicle
16. Compound Finance, "Price Feed" documentation.
    https://compound.finance/docs/prices
17. RedStone, "Blockchain Oracles Comparison: Chainlink vs Pyth vs
    RedStone 2026," Mar 2026.
    https://blog.redstone.finance/2026/03/30/blockchain-oracles-comparison-chainlink-vs-pyth-vs-redstone-2026/
18. Liquity, "The Oracle Conundrum," 2023.
    https://www.liquity.org/blog/the-oracle-conundrum
19. Wormhole, "VAAs (Verifiable Action Approvals)" documentation,
    accessed Apr 2026.
    https://wormhole.com/docs/protocol/infrastructure/vaas/
20. Wormhole, "Guardians" documentation, accessed Apr 2026.
    https://wormhole.com/docs/protocol/infrastructure/guardians/
21. Wormhole, "Security" documentation, accessed Apr 2026.
    https://wormhole.com/docs/protocol/security/
22. Sec3, "How Do Cross-Chain Bridges Work? A Case on Wormhole
    (Part 2)," 2022.
    https://www.sec3.dev/blog/bridges2
23. Ethereum, "Precompiled Contracts" (ECRECOVER at address 0x01,
    base cost 3,000 gas).
    https://www.evm.codes/precompiled
24. Solana Labs, "block_cost_limits.rs" (secp256k1 verification
    compute units).
    https://github.com/solana-labs/solana/blob/master/cost-model/src/block_cost_limits.rs
25. Solana Foundation, "Fee Structure" (Ed25519, Secp256k1,
    Secp256r1 precompile programs).
    https://solana.com/docs/core/fees/fee-structure
26. Pyth Network, "Perseus Network Upgrade" blog post, 2025.
    https://www.pyth.network/blog/perseus-network-upgrade
27. Wormhole, "Boundless Partners with Wormhole to Launch ZK
    Network Powered by RISC Zero," 2025.
    https://wormhole.com/blog/boundless-partners-with-wormhole-to-launch-zk-network-powered-by-risc-zero
28. RedStone, "How Data Flows to the Blockchain" architecture
    documentation, accessed Apr 2026.
    https://docs.redstone.finance/docs/architecture/
29. RedStone, "rust-sdk" repository (crates/redstone, including
    crypto/mod.rs and Cargo.toml dependencies on secp256k1 and
    k256).
    https://github.com/redstone-finance/rust-sdk
30. RedStone, "redstone-oracles-monorepo" EVM connector
    (RedstoneConsumerBase.sol, getUniqueSignersThreshold,
    SignatureLib.recoverSignerAddress).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/evm-connector/contracts/core/RedstoneConsumerBase.sol
31. Sui Documentation, "Module sui::ecdsa_k1"
    (secp256k1_ecrecover Move builtin).
    https://docs.sui.io/references/framework/sui_sui/ecdsa_k1
32. Stellar, "CAP-0051: Smart Contract Host Functionality"
    (recover_key_ecdsa_secp256k1, 2.3M CPU instructions).
    https://github.com/stellar/stellar-protocol/blob/master/core/cap-0051.md
33. Veridise, "RedStone Stellar Connector" security assessment,
    Oct 2025.
    https://veridise.com/wp-content/uploads/2025/10/VAR-Redstone-251006-Oracles-SDK-V2.pdf
34. RedStone, "redstone-oracles-monorepo" Fuel connector
    (Sway contract adapter).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/fuel-connector/README.md
35. RedStone, "RedStone Brings Secure, Gas-Efficient Oracle
    Solutions to Radix DeFi Ecosystem," Jun 2025.
    https://blog.redstone.finance/2025/06/12/redstone-brings-secure-gas-efficient-oracle-solutions-to-radix-defi-ecosystem/
36. RedStone, "redstone-oracles-monorepo" Stellar connector
    DeployingFeed.md (Stellar account key uses Ed25519-curve;
    refers to deployer key, not RedStone signing scheme).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/570006fccf0f919ad9722d11914dd0bc1c5b136d/packages/stellar-connector/DeployingFeed.md
37. CoinDesk, "Bitcoin Flash Crashed to $5K on Pyth Network's Data
    Feed," Sep 2021.
    https://www.coindesk.com/markets/2021/09/22/bitcoin-flash-crashes-to-5k-on-pyth-networks-data-feed
38. Morpho governance forum, "PYTH CBETH price feed is easily
    manipulated, resulted in me losing $33,000," Mar 2025.
    https://forum.morpho.org/t/pyth-cbeth-price-feed-is-easily-manipulated-resulted-in-me-losing-33000/1577
39. Chainlink, XMR/USD price feeds: Optimism Mainnet
    (`0x2a8D91686A048E98e6CCF1A89E82f40D14312672`) and Polygon
    Mainnet (`0xBE6FB0AB6302B693368D0E9001fAF77ecc6571db`); Data
    Streams XMR/USD-RefPrice product on 43+ chains.
    https://data.chain.link/feeds/optimism/mainnet/xmr-usd
    https://data.chain.link/feeds/polygon/mainnet/xmr-usd
    https://data.chain.link/streams/xmr-usd-cexprice-streams
40. Chainlink, ZEC/USD price feeds: Ethereum Mainnet
    (`0x3f929667bdf783b99274F10465a89d6aF772736E`) and Polygon
    Mainnet (`0xBC08c639e579a391C4228F20d0C29d0690092DF0`); Data
    Streams ZEC/USD-RefPrice product on 35+ chains.
    https://data.chain.link/ethereum/mainnet/crypto-usd/zec-usd
    https://data.chain.link/feeds/polygon/mainnet/zec-usd
    https://data.chain.link/streams/zec-usd-cexprice-streams
41. Pyth Network, legacy price feeds dashboard
    (`crypto-xmr-usd`, `crypto-zec-usd`).
    https://insights.pyth.network/legacy-price-feeds/crypto-xmr-usd
    https://insights.pyth.network/legacy-price-feeds/crypto-zec-usd
42. RedStone, "ALL_SUPPORTED_TOKENS" registry (XMR and ZEC listed).
    https://github.com/redstone-finance/redstone-api/blob/main/docs/ALL_SUPPORTED_TOKENS.md
43. DIA, asset price index (XMR and ZEC).
    https://www.diadata.org/app/price/asset/Monero/0x0000000000000000000000000000000000000000/
    https://www.diadata.org/app/price/asset/Zcash/0x0000000000000000000000000000000000000000/
44. Supra, "Data Feeds Index" (XMR_USDT, ZEC_USDT).
    https://docs.supra.com/oracles/data-feeds/data-feeds-index
