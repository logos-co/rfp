# Appendix: Oracle Ecosystem

This appendix surveys oracle protocols, TWAP mechanics, manipulation
vectors, and external oracle models relevant to
[RFP-018](../RFPs/RFP-018-twap-oracle.md). It provides the technical
and market context for the two-tier oracle architecture proposed for
LEZ.

## Oracles Surveyed

Protocols are ordered by Total Value Secured (TVS), largest first.
This order is maintained throughout the document.

| Protocol | TVS | Chains | Model | Feed Count | Key Feature |
|----------|-----|--------|-------|------------|-------------|
| Chainlink | $93B+ | 27 push / 100+ pull | Push (OCR/DON) | 1,000+ | Decentralised Oracle Network with VWAP from premium data aggregators |
| Chronicle | $9.2B+ | 13 | Push | Limited | MakerDAO-native; concentrated TVS from Sky's $9B+ TVL |
| Pyth | $8.6B+ | 81 | Pull (Wormhole) | 1,500+ | First-party data from 70+ institutional publishers; confidence intervals |
| RedStone | $7.2B+ | 50+ push / 110+ pull | Pull (calldata) | 1,000+ | No bridge dependency; modular push+pull; fastest-growing oracle |
| Switchboard | $3B+ | 9 | Pull (TEE) | Permissionless | TEE (SGX/SEV) security; permissionless custom feed creation |
| Supra | $650M+ | 45 | Push+Pull | N/A | Newer entrant; DORA (Distributed Oracle Agreement) consensus |

## Scale and Traction

Total Value Secured measures the aggregate DeFi TVL in protocols that
depend on a given oracle. An oracle with higher TVS has withstood
more economic scrutiny, though TVS is a trailing indicator: new chains
start at $0 regardless of oracle choice.

### Market share

Chainlink dominates with approximately 68% of global oracle TVS and
84% on Ethereum specifically [1]. Chronicle's high TVS (approximately
6.7%) is concentrated in a single protocol (MakerDAO/Sky) and does
not reflect general-purpose adoption [1]. Pyth (approximately 6.3%)
leads on chain breadth (81+ chains) and data quality through
first-party institutional publishers. RedStone (approximately 5.2%)
has the fastest growth trajectory, driven by explicit support for L2s,
appchains, and rollups [2][3].

### Per-protocol adoption

**Chainlink.** Deployed on 27 chains with push model and 100+ via
CCIP. Integrates with 452 protocols. Requires 3+ independent RPC
providers with 99.9% uptime SLAs before deploying to a new chain,
making it inaccessible for new chains at launch [4]. Uses Off-Chain
Reporting (OCR) where nodes aggregate prices from premium data vendors
(Kaiko, CoinMetrics) and submit a single signed transaction per round.
Update triggers are deviation threshold (e.g. 0.5% price change) or
heartbeat (e.g. 1 hour maximum staleness).

**Pyth.** Originated on Solana and now cross-chain via Wormhole.
Aggregates first-party data from 70+ publishers including Jane Street,
Coinbase, and Binance [5]. Architecture: publishers post prices on
Pythnet (a Solana appchain) every 400ms; Wormhole guardians sign a
Merkle root; the Hermes off-chain cache stores latest proofs; users
fetch and submit proofs in their transactions. Every price includes a
confidence interval, enabling protocols to reject high-uncertainty
updates. Known incidents include an 87% BTC mispricing (2021), a
40-minute downtime on BTC and 8 other feeds, and wrongful liquidations
on Morpho (Base) and Jupiter (Solana) [5].

**RedStone.** Modular oracle with both push and pull delivery. Pull
model attaches signed data to EVM calldata; the on-chain contract
verifies node signatures without requiring a bridge or dedicated
relay infrastructure [3]. Fastest-growing oracle in 2024 to 2025,
with deployments on Monad, Hyperliquid (HyperStone), and 110+ pull
chains. Zero reported mispricing incidents as of early 2026 [3].
Expanding into RWA feeds (BlackRock BUIDL, VanEck VBILL) and risk
ratings via Credora acquisition.

**Switchboard.** Permissionless feed creation via TEE (SGX/SEV)
oracle nodes [6]. Any developer can create custom feeds for assets
not covered by Pyth or RedStone. Can aggregate from multiple upstream
oracles (Pyth + Chainlink + custom APIs) in a single feed. The core
EVM contract must be deployed by the Switchboard team (not
self-serve), and TEE hardware requirements limit the operator pool.

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
using the price set by the last trade of the previous block [8]. The
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
which contaminates the accumulator sample in block N+1 [4][10].

### PoS multi-block validator attacks

Under Proof of Stake, validators know one epoch ahead (32 blocks on
Ethereum, approximately 6.4 minutes) whether they control consecutive
blocks. A validator controlling two consecutive blocks can move the
price in block N and reverse it in block N+1, at a cost of only 2x
pool fees, with no back-run competition [4]. This contaminates one
accumulator data point per attack. On high-liquidity pools (e.g.
USDC/WETH 5bps on Ethereum), the attack is economically infeasible;
on low-liquidity pools, it is trivially cheap.

### Low-liquidity vulnerability

Manipulation cost scales approximately linearly with pool liquidity
depth [4][10]. Adding $1M of wide-range liquidity on the USDC/WETH
5bps pool increases the two-block attack cost by approximately $360B,
demonstrating extreme sensitivity [4]. On a new chain like LEZ with
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

1. DefiLlama, "Oracles" dashboard, accessed Q3 2025.
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
