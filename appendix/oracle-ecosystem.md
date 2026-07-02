# Appendix: Oracle Ecosystem

This appendix surveys oracle protocols, TWAP mechanics, manipulation vectors,
and external oracle models relevant to
[RFP-019](../RFPs/RFP-019-twap-oracle.md). It provides the technical and market
context for the two-tier oracle architecture proposed for LEZ.

## Oracles Surveyed

DeFi-style oracles are ordered by Total Value Secured (TVS), largest first; this
order is maintained throughout the document. The DLC-oracle row is appended at
the end because the DLC attestation model does not have a TVS metric comparable
to push/pull DeFi oracles (DLC oracles secure individual Bitcoin DLC contracts
at maturity rather than continuously-running DeFi positions). DLC oracles are
included because the BIP-340 attestation format they publish is the LEZ-native
signature primitive, which makes them relevant to the verification-cost analysis
later in this document; their structural fit is prediction markets, not
streaming price feeds.

| Protocol                                                                                                                                 | TVS                                              | Chains                                                               | Model                                                                 | Feed Count                            | Key Feature                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Chainlink                                                                                                                                | $66B-$75B (May 2025)                             | 27 push / 60+ via CCIP                                               | Push (OCR/DON)                                                        | 1,000+                                | Decentralised Oracle Network with VWAP from premium data aggregators                                                                                                                                                           |
| Chronicle                                                                                                                                | $10.2B+ (per [2])                                | 13                                                                   | Push                                                                  | Limited                               | MakerDAO-native; concentrated TVS from Sky's $10B+ TVL                                                                                                                                                                         |
| Pyth                                                                                                                                     | $8.6B+                                           | 50+ via Wormhole                                                     | Pull (Wormhole)                                                       | 2,800+                                | First-party data from 120+ institutional publishers; confidence intervals                                                                                                                                                      |
| RedStone                                                                                                                                 | $10B+                                            | 50+ push / 120+ pull                                                 | Pull (calldata)                                                       | 1,000+                                | No bridge dependency; modular push+pull; fastest-growing oracle                                                                                                                                                                |
| Switchboard                                                                                                                              | $3B+ [1]                                         | 9                                                                    | Pull (TEE)                                                            | Permissionless                        | TEE (SGX/SEV) security; permissionless custom feed creation                                                                                                                                                                    |
| Supra                                                                                                                                    | $650M+ [1]; Supra positioning cites 50+ networks | 45                                                                   | Push+Pull                                                             | N/A                                   | Newer entrant; DORA (Distributed Oracle Agreement) consensus                                                                                                                                                                   |
| DLC oracles (Pythia live; Sibyls operator wound down and repo deleted; P2PDerivatives dormant; Ernest on hiatus; Magnolia closed source) | N/A (not DeFi-TVS measured)                      | Bitcoin native; BIP-340 attestations portable to any verifying chain | Event-driven attestation (pre-announced R-points, signed at maturity) | Limited (BTC/USD; some chain metrics) | Native BIP-340 Schnorr; live ecosystem split between plain SHA-256 (Pythia, P2PDerivatives, rust-dlc) and tagged SHA-256 (Kormir, Ernest, Sibyls dlc_v0 mode); structural fit is prediction markets, not streaming price feeds |

## Scale and Traction

Total Value Secured measures the aggregate DeFi TVL in protocols that depend on
a given oracle. An oracle with higher TVS has withstood more economic scrutiny,
though TVS is a trailing indicator: new chains start at $0 regardless of oracle
choice.

### Market share

Chainlink dominates with approximately 68% of global oracle TVS and over 80% on
Ethereum specifically [1]. Chronicle's high TVS (approximately 17%) is
concentrated in a single protocol (MakerDAO/Sky) and does not reflect
general-purpose adoption [2]. Pyth leads on data quality through first-party
institutional publishers; per Pyth's own April 2026 positioning, the network
reaches "100+ blockchains" via Wormhole, with cross-chain support established on
roughly 50+ chains in production. RedStone has the fastest growth trajectory,
driven by explicit support for L2s, appchains, and rollups. Per RedStone's own
reporting [3][17], no oracle-induced mispricing has occurred on its flagship
integrations (Ethena, Gearbox) through early 2026; no independent post-incident
review corroborates this absence claim.

### Per-protocol adoption

**Chainlink.** Deployed on 27 chains with push model and 60+ public and private
blockchains via CCIP. Network integration requirements include high-availability
RPC providers, valid SSL, Ethereum JSON-RPC compatibility, and 30-day historical
RPC performance metrics, making it inaccessible for new chains at launch [4].
Uses Off-Chain Reporting (OCR) where nodes aggregate prices from premium data
vendors (Kaiko, CoinMetrics) and submit a single signed transaction per round.
Update triggers are deviation threshold (e.g. 0.5% price change) or heartbeat
(e.g. 1 hour maximum staleness).

**Pyth.** Originated on Solana and now cross-chain via Wormhole. Aggregates
first-party data from 120+ institutional publishers (April 2026), including Jane
Street, Coinbase, and Binance, across 2,800+ price feeds (December 2025).
Architecture: publishers post prices on Pythnet (a Solana appchain) every 400ms;
Wormhole guardians sign a Merkle root; the Hermes off-chain cache stores latest
proofs; users fetch and submit proofs in their transactions. Every price
includes a confidence interval, enabling protocols to reject high-uncertainty
updates. Known incidents include the September 2021 BTC/USD mispricing, where
the feed reported approximately $5,402 against an actual price near $43,500 (an
approximately 87% drop) [37], and the March 2025 Morpho cbETH wrongful
liquidation on Base, where a roughly 7-minute cbETH/ETH staleness window
distorted the collateral price and liquidated a user for approximately $33,000
on a Re7 Labs vault [38].

**RedStone.** Modular oracle with both push and pull delivery. Pull model
attaches signed data to EVM calldata; the on-chain contract verifies node
signatures without requiring a bridge or dedicated relay infrastructure [3].
Fastest-growing oracle in 2024 to 2025, with deployments on Monad, Hyperliquid
(HyperStone), and 120+ pull chains. Per RedStone's own reporting [3][17], zero
mispricing incidents through early 2026; no independent post-incident review
corroborates this. Expanding into RWA feeds (BlackRock BUIDL, VanEck VBILL) and
risk ratings via Credora acquisition.

**Switchboard.** Permissionless feed creation via TEE (SGX/SEV) oracle nodes
[13]. Any developer can create custom feeds for assets not covered by Pyth or
RedStone. Can aggregate from multiple upstream oracles (Pyth + Chainlink +
custom APIs) in a single feed. The core protocol contract is a single-deployment
artefact per chain (deployed by the Switchboard team), but feed creation on top
of it is permissionless; TEE hardware requirements limit the operator pool.

**DIA Lumina.** Fully permissionless: both data sourcing (Feeder nodes) and feed
deployment (Aggregator contracts) require no team permission [7]. ZK proof
verification aligns with privacy-focused chains. Smaller feed catalogue (200+)
and newer ZK architecture (Lumina V2) make it less battle-tested than Pyth or
RedStone.

## TWAP Mechanics

### Uniswap v2 accumulator

Each Uniswap v2 pool stores `price0CumulativeLast` and `price1CumulativeLast`
variables. These accumulators are updated at the beginning of each block (before
any same-block trades execute) using the price set by the last trade of the
previous block [14]. The formula: `cumulativePrice += price * timeElapsed`. To
compute a TWAP, an external contract reads the accumulator at two timestamps (T1
and T2) and divides:

```
TWAP = (accumulator[T2] - accumulator[T1]) / (T2 - T1)
```

This yields an arithmetic mean price. No off-chain components are required.

### Uniswap v3 accumulator

Uniswap v3 replaced the raw price accumulator with a tick-based accumulator
(`tickCumulative`), which stores the running sum of
`currentTick * secondsElapsed` [9]. The TWAP is:

```
averageTick = (tickCumulative[T2] - tickCumulative[T1]) / (T2 - T1)
price = 1.0001^averageTick
```

This yields a geometric mean price, which is more appropriate for multiplicative
price processes. The observation buffer is a circular array of up to 65,535
slots, expandable via `increaseObservationCardinalityNext()` at a one-time gas
cost [9].

### Uniswap v4 truncated oracle hook

Uniswap v4 removed the oracle from core pool state and moved it to an optional
hook, so pools that do not need an oracle no longer pay the per-swap
accumulator-update gas. The v3 design described above remains the production
reference for on-chain TWAP (v3 is still the dominant AMM by deployed volume),
and the v4 hook builds on the same tick-accumulator pattern rather than
replacing it.

The notable evolution in v4 is the **truncated oracle hook**, which caps the
per-block tick movement that the accumulator records [61]. Before each swap, the
hook compares the pool's current tick against the previous-block tick stored by
the hook. If the absolute difference exceeds a threshold (the Uniswap reference
implementation uses 9,116 ticks, corresponding to roughly a 2.49x price move per
block: `1.0001^9116 ≈ 2.49`), the value written to the accumulator is clamped to
±threshold rather than the raw observed tick. The geometric-mean TWAP formula is
otherwise identical to v3.

Concretely:

```
observedTick   = pool.currentTick
previousTick   = hook.lastObservedTick
delta          = observedTick - previousTick
clampedDelta   = clamp(delta, -MAX_TICK_DELTA, +MAX_TICK_DELTA)
recordedTick   = previousTick + clampedDelta
tickCumulative += recordedTick * timeElapsed
```

The effect is that a single-block manipulation can shift the accumulator by at
most `MAX_TICK_DELTA * timeElapsed`, regardless of how far the attacker actually
moved the pool. To produce the same oracle distortion that an unbounded v3
accumulator would record from one manipulated block, an attacker must sustain a
manipulated price across many consecutive blocks (Uniswap's reference scenario
cites roughly 15 blocks for the oracle to "catch up" to a large move), during
which arbitrage bots compete on the unclamped pool price and erode the
attacker's position [61]. The truncated hook does not eliminate the
cost-vs-window tradeoff (long sustained manipulation on a thin pool is still
feasible), but it raises the per-block cost floor and is currently the strongest
in-AMM mitigation against single-block and PoS multi-block oracle attacks (see
"TWAP Manipulation Vectors" below).

For LEZ, this is directly relevant: an LEZ-side on-chain TWAP that emulates v3's
raw accumulator inherits v3's vulnerability to thin pools at chain launch.
Porting the truncation logic is a low-cost addition (one tick-delta comparison
and clamp per accumulator update) that materially raises the manipulation floor
without requiring an external feed.

### Geometric vs arithmetic mean

The arithmetic mean (v2) is sensitive to outliers: an attacker who moves the
price up by 10x in one block and back by 10x in the next leaves a net upward
bias in the arithmetic mean. The geometric mean (v3) is invariant to such
symmetric multiplicative manipulation, because `log(10x) + log(1/10x) = 0` [9].
For this reason, the v3 approach is strictly preferred for price oracle use.

### Example computation (v3)

```
tickCumulatives = [70,000, 1,070,000] over 10 seconds
averageTick = (1,070,000 - 70,000) / 10 = 100,000
price = 1.0001^100,000 = 22,015.5 USDC/WETH
```

Minor imprecision arises because ticks are integers and fractional ticks are
truncated, but this is negligible for most use cases [9].

## TWAP Manipulation Vectors

### Flash loan attacks (within-block)

An attacker borrows massive capital via a flash loan, moves the pool price
heavily, and profits from a protocol that reads the manipulated price in the
same or next block. Under Uniswap v2/v3, the accumulator is updated at block
start (before same-block trades), so within-block manipulation does not affect
the current block's accumulator sample. However, the attacker can manipulate
price at the end of block N, which contaminates the accumulator sample in block
N+1 [6][10].

### PoS multi-block validator attacks

Under Proof of Stake, validators know one epoch ahead (32 blocks on Ethereum,
approximately 6.4 minutes) whether they control consecutive blocks. A validator
controlling two consecutive blocks can move the price in block N and reverse it
in block N+1, at a cost approximately equal to the round-trip swap fees and
price impact, with no back-run competition [6]. This contaminates one
accumulator data point per attack. On high-liquidity pools (e.g. USDC/WETH 5bps
on Ethereum), the attack is economically infeasible; on low-liquidity pools, it
is trivially cheap.

### Low-liquidity vulnerability

Manipulation cost scales approximately linearly with pool liquidity depth
[6][10]. On the deep USDC/WETH 5bps pool, adding $1M of wide-range liquidity
raises the marginal cost of a two-block oracle attack by approximately $360B
above the already-large baseline, demonstrating extreme sensitivity [6]. On a
new chain like LEZ with $1M pools, the same attack that costs trillions on
Ethereum mainnet costs only thousands.

### Short observation window attacks

An attacker sustains a manipulated price across multiple blocks for the duration
of a short TWAP window (e.g. 5 minutes). This is more expensive than
single-block manipulation but feasible on low-liquidity pools [10]. The cost
scales with both window length and pool depth.

### Mitigation: tick-delta truncation (Uniswap v4)

The Uniswap v4 truncated oracle hook (described above under "Uniswap v4
truncated oracle hook") clamps per-block tick movement written to the
accumulator [61]. This directly raises the cost of flash-loan and PoS
multi-block attacks: a single manipulated block contributes at most
`MAX_TICK_DELTA * blockTime` to the accumulator regardless of the actual price
excursion, forcing an attacker to sustain manipulation across many blocks and
pay arbitrage costs each block. It does not address the
short-window-on-thin-pool case, where the attacker is willing to sustain price
across the full window; defending that case still requires either deeper
liquidity or a divergence check against an external feed.

### Historical losses

According to the Rekt Database (cited in the Ormer paper), 36 flash loan oracle
attacks alone caused over $418M in cumulative losses [5]. Oracle manipulation is
the primary attack vector in DeFi exploits.

## Window Selection Tradeoffs

### The core tradeoff

Short TWAP windows (e.g. 5 minutes) provide fresh prices but are cheap to
manipulate. Long windows (e.g. 24 hours) are expensive to manipulate but lag the
market severely during genuine volatility. Unlike external oracle networks, AMM
TWAP cannot simultaneously optimise both security and freshness [10].

### Production standards

The majority of TWAP oracles in production DeFi use windows between 30 minutes
and 12 hours [11].

| Use case                     | Typical window | Rationale                                                    |
| ---------------------------- | -------------- | ------------------------------------------------------------ |
| Lending collateral valuation | 30 min to 2 h  | Resist short-term manipulation; liquidation timing tolerable |
| DEX AMM internal pricing     | 5 to 30 min    | Requires responsiveness for arbitrage                        |
| Governance and voting        | 24 h to 7 days | Resist flash attacks on governance weight                    |

### Cardinality math

At 12-second blocks, the maximum observation cardinality of 65,535 provides
approximately 218 hours (roughly 9 days) of lookback [9]. Protocols trading off
storage cost vs lookback depth should expand cardinality to at least
`(desired_window_seconds / block_time) + buffer`. For a 2-hour window at 12s
blocks, minimum cardinality is 600 + buffer.

### Manipulation cost scaling

Moving the price by 5% on a 1-hour TWAP requires sustaining that 5% deviation
every block for 1 hour, at a cost approximately equal to the arbitrage losses
and fees incurred per block, multiplied by the number of blocks [10]. Doubling
pool liquidity approximately doubles the cost; doubling the window length
approximately doubles the cost.

## External Oracle Models

### Push vs pull

Push oracles (Chainlink) submit price updates to an on-chain contract on a
heartbeat (e.g. every hour) or deviation trigger (e.g. 0.5% price change). Gas
cost per update ranges from 300K to 2.4M gas depending on the aggregation method
[4]. Pull oracles (Pyth, RedStone) sign data off-chain; the consumer fetches the
signed data and submits it as part of their own transaction, paying the
verification gas cost (approximately 50K to 100K gas per update) [3][5].

For new chains, pull is strongly preferred: no dedicated per-chain node
operators, no ongoing gas subsidies, and immediate availability once the
verification contract is deployed.

### DON vs single-source

Chainlink's Decentralised Oracle Network (DON) uses Off-Chain Reporting (OCR)
where multiple independent nodes each fetch prices from premium data
aggregators, communicate via P2P, elect a leader who produces a signed report
containing all observations, and submit the median on-chain [12]. This is
technically VWAP (volume-weighted from multi-exchange aggregation), not a strict
TWAP from a single AMM [12].

Single-source oracles (a single AMM TWAP) offer maximum trustlessness but depend
entirely on that source's liquidity and availability. DON-style oracles trade
some trustlessness (off-chain node operators) for dramatically better
manipulation resistance and market coverage.

### Comparison table

| Dimension               | AMM TWAP                    | Pyth                                          | RedStone                             | Chainlink DON                                                                                                                   |
| ----------------------- | --------------------------- | --------------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| Trust model             | Trustless (on-chain)        | Semi-trusted (publishers + Wormhole)          | Semi-trusted (node signatures)       | Semi-trusted (node operators + staking)                                                                                         |
| Data source             | Single DEX pool             | 70+ first-party publishers                    | CEX + DEX + aggregators              | Premium aggregators (Kaiko, CoinMetrics)                                                                                        |
| Market coverage         | On-chain pairs only         | 2,800+ feeds (Dec 2025; crypto, FX, equities) | 1,000+ feeds                         | 1,000+ feeds                                                                                                                    |
| New-chain deployability | Requires AMM with liquidity | Requires Wormhole                             | No bridge needed                     | Requires multiple high-availability RPC providers, valid SSL, JSON-RPC compatibility, 30-day historical RPC performance metrics |
| Manipulation resistance | Scales with pool depth      | Independent of on-chain liquidity             | Independent of on-chain liquidity    | Independent of on-chain liquidity                                                                                               |
| Gas per query           | Very low (read accumulator) | 50K to 100K (VAA verification)                | 50K to 100K (signature verification) | N/A (push: consumer reads storage)                                                                                              |
| Confidence interval     | No                          | Yes                                           | No                                   | No                                                                                                                              |
| Real-world assets       | No                          | Yes                                           | Yes (RWA feeds)                      | Yes                                                                                                                             |

## Infrastructure Requirements for External Oracles on LEZ

The architectural choice (push vs pull, bridge vs direct signature) determines
what Logos has to deploy, host, or coordinate to make each external oracle
available on LEZ. This section is operational: it answers "who runs what, where
does the data come from, and what happens if it breaks." It complements the
higher-level model comparison above.

The summary up front: **RedStone is a self-deploy task** (port a verifier
program, optionally run a relayer); **Pyth is a multi-party governance task**
gated on Wormhole guardian-set support for LEZ; **Chainlink is impractical at
launch** (DON onboarding has no public path for new chains without months of
integration work and high-availability RPC providers in place).

### Side-by-side: bringing an external oracle live on LEZ

| Dimension                         | RedStone (Pull)                                                                                                                                                                                                             | RedStone (Push)                                                                                                     | Pyth                                                                                                                              | Chainlink                                                                               |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Relayer service required?         | No (user attaches signed payload to their tx)                                                                                                                                                                               | Yes (a service writes prices on-chain on a schedule)                                                                | No scheduled relayer (Hermes is a read-only cache)                                                                                | Yes (DON node operators write on-chain)                                                 |
| Off-chain data endpoint           | RedStone DDL HTTPS gateways                                                                                                                                                                                                 | Same                                                                                                                | Hermes HTTPS REST/SSE                                                                                                             | DON gossip + private aggregator data                                                    |
| Bridge / cross-chain trust layer  | None: signatures verified directly                                                                                                                                                                                          | None                                                                                                                | Wormhole (19 guardians, 13/19 threshold)                                                                                          | None for the price flow itself, but DON nodes coordinate via Wormhole-free P2P          |
| New-chain onboarding type         | Self-deploy: any team ports the verifier                                                                                                                                                                                    | Self-deploy: same plus a relayer                                                                                    | Multi-party: Wormhole governance proposal, Core Contract port, guardian observation, Pyth Receiver port                           | Multi-party: Chainlink integration program, DON spin-up, RPC provider qualification [4] |
| Contracts to deploy on LEZ        | One verifier module per consumer (or one shared `ConsumerBase`-equivalent)                                                                                                                                                  | Same plus `PriceFeedsAdapter`-equivalent                                                                            | Wormhole Core port + Pyth Receiver port                                                                                           | DON aggregator contract per feed                                                        |
| Who runs the off-chain infra      | RedStone (default DDL gateways)                                                                                                                                                                                             | RedStone, Logos ecosystem, or a consumer protocol can run relayer instances                                         | Pyth Data Association (default Hermes) or third-party node providers (Triton, P2P, extrnode, Liquify) [62]                        | Chainlink + node operator coalition                                                     |
| Gas paid by                       | Consumer on every read (calldata + verification)                                                                                                                                                                            | Relayer wallet, per update                                                                                          | Consumer on every update (VAA verification + Merkle proof)                                                                        | Effectively the consumer (heartbeat cost socialised across all readers)                 |
| Logos ongoing operational burden  | None on LEZ side                                                                                                                                                                                                            | Relayer instances + signing wallet gas + monitoring                                                                 | Optional: self-hosted Hermes mirror for low latency; otherwise none on LEZ side                                                   | Continuous: DON nodes, RPC infrastructure, integration support                          |
| Time-to-live for a new chain      | Weeks (verifier port + audit)                                                                                                                                                                                               | Same                                                                                                                | Months (Wormhole governance + cross-runtime port)                                                                                 | Months to quarters, gated on Chainlink integration program                              |
| Trust assumptions added           | RedStone authorised signer set (~3-of-N per `dataServiceId`); RedStone DDL HTTPS gateways for liveness (censorship/outage; signatures still protect against falsified data — see "HTTPS portals are a centralisation risk") | Same plus the relayer (only for liveness; data is still signature-verified on-chain); DDL gateway liveness as above | Pythnet validators + 13-of-19 Wormhole guardians + publisher set; Hermes HTTPS gateway for liveness (same caveat as RedStone DDL) | Chainlink node operator set + premium aggregator licensing chain                        |
| Documented LEZ-relevant precedent | None for LEZ-native SVM runtime; EVM port pattern is mature                                                                                                                                                                 | Same                                                                                                                | No SVM-non-Solana port of the Pyth receiver is documented as of May 2026                                                          | No new-chain integration without Chainlink-side prioritisation                          |

### Where the signed data comes from

#### RedStone

The signed data lives in RedStone's **Data Distribution Layer (DDL)**, a set of
public HTTPS gateways operated by RedStone. The SDK and any relayer fetch from
URLs hardcoded in
[`packages/sdk/src/data-services-urls.ts`](https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/sdk/src/data-services-urls.ts)
\[63\]:

- `https://oracle-gateway-1.a.redstone.finance` (AWS, production, metadata, no
  historical)
- `https://oracle-gateway-2.a.redstone.finance` (AWS, production, metadata,
  historical)
- `https://oracle-gateway-1.a.redstone.vip` (GCP, production, no metadata, no
  historical)

A request specifies a `dataServiceId` (e.g. `redstone-primary-prod`), the feed
IDs to fetch (e.g. `BTC`, `ETH`), and the minimum number of unique signers
required (typically 3). The response is a signed data package: per signer, a
payload over `(feedId, value, timestamp)` plus an ECDSA signature, with packages
concatenated. No authentication on the read paths.

#### Pyth

The signed data is exposed via **Hermes**, an HTTPS REST + SSE service that
mirrors Wormhole-signed Pyth Merkle roots \[62\]:

- `https://hermes.pyth.network` (public, operated by Pyth Data Association, 20
  req / 10 s per IP)
- Third-party node providers for production: Triton, P2P, extrnode, Liquify [62]

The v2 API exposes `/v2/updates/price/latest`,
`/v2/updates/price/{publish_time}`, and an SSE stream at
`/v2/updates/price/stream`. The payload is a binary VAA (base64 or hex),
submittable directly as calldata to the on-chain Pyth Receiver contract.

### HTTPS portals are a centralisation risk

Both RedStone DDL and Pyth Hermes are **HTTPS gateways**, not P2P networks.
There is no gossip layer, no DHT, no decentralised fan-out: every consumer of
signed data fetches from a small set of DNS-resolved endpoints controlled by the
oracle provider or one of its named third-party hosts. This has three direct
consequences:

- **Censorship**. The operator (and the network operator upstream) can withhold
  responses to specific clients, geographies, or asset IDs. Signature
  verification still protects against *falsified* data, but does not protect
  against *withheld* data: a censored consumer simply cannot fetch the latest
  update and their transaction will revert.
- **Outage**. DNS resolution, TLS certificate validity, and HTTP availability
  all become liveness dependencies. A regional cloud outage at AWS (most
  RedStone gateways) or at the Pyth Data Association's Hermes hosting takes the
  oracle offline for affected users.
- **Observability**. The gateway sees who is reading what. Even though Logos's
  confidential execution model keeps the on-chain *use* of a price private, the
  off-chain *fetch* of a signed payload is fully observable to the gateway
  operator.

These are inherent properties of the HTTPS-portal design, not implementation
bugs. They are present on every chain that uses RedStone or Pyth today; LEZ
inherits them by adopting either oracle. Mitigations available to a consumer:

- Run a local mirror of the gateway (RedStone supports
  `OVERRIDE_DIRECT_CACHE_SERVICE_URLS`; Pyth supports running a private Hermes
  instance against Pythnet + Wormhole feeds).
- Query multiple gateway hosts in parallel (the RedStone SDK default; Pyth
  recommends third-party node providers).
- Maintain a recent-price fallback on-chain (last verified update served until a
  new one arrives), accepting staleness risk for liveness.

None of these mitigations eliminate the centralisation; they reduce specific
failure modes at the cost of additional infrastructure or risk acceptance.

### RedStone relayer setup (Push mode), end to end

The reference relayer is open source in
[`redstone-finance/redstone-oracles-monorepo`](https://github.com/redstone-finance/redstone-oracles-monorepo),
Node.js/TypeScript. Per cycle it:

1. Polls a DDL gateway via the SDK for signed packages matching the configured
   `dataServiceId` and feed IDs.
2. Compares against the last on-chain value. Writes a new transaction when
   either trigger fires: heartbeat (`UPDATE_PRICE_INTERVAL`, e.g. 5 min) or
   deviation (`MIN_DEVIATION_PERCENTAGE`, e.g. 0.5%).
3. Packs the concatenated signed packages as transaction data targeting the
   on-chain adapter; signs with the relayer's hot wallet (`PRIVATE_KEY` env
   var).
4. Submits, monitors, repeats.

The on-chain adapter, on receipt, verifies each signer's signature against the
known signer set, checks the package timestamp is fresh (within a configurable
window), takes the median across signers, and writes the result.

For LEZ specifically, three artefacts are needed beyond the RedStone reference
code:

- A SPEL port of the `PriceFeedsAdapter` writing to the canonical price account
  from RFP-019 (populating `base_asset`, `quote_asset`, price, timestamp,
  source, confidence).
- A SPEL port of the consumer-base library if Pull mode is offered alongside
  Push.
- LEZ-specific transaction-building and signing bindings in the relayer (most
  likely via the same SDK that consumer programs use).

Operational footprint per Push deployment:

- 1+ relayer process (a few hundred MB RAM each; cloud or bare metal). RedStone
  recommends 2+ independent instances for redundancy.
- 1 hot wallet on LEZ per instance, funded with gas.
- Monitoring: feed freshness on-chain, wallet balance, gateway reachability,
  transaction failure rate.

Typical per-update verification cost on EVM is 100-200k gas; the LEZ-side
equivalent under SVM CU pricing should fall in the same range as the pull-mode
verification cost documented in the
[Signature Verification Schemes](#signature-verification-schemes) section, since
the verification work is identical and only the cost allocation differs (relayer
vs consumer).

### Who runs the relayer

RedStone's documentation says the relayer is permissionless: "anyone could run
the service as the data is eventually validated on-chain" [64]. In practice on a
new chain, the operating party is usually one of:

1. **RedStone itself**, as part of an integration package. Monad mainnet (24 Nov
   2025\) is the recent precedent: RedStone shipped 50+ feeds and ran the
   infrastructure from day one [65].
2. **The chain ecosystem**, funding and running the relayer as shared
   infrastructure.
3. **A consuming protocol**, if it wants exclusive control over its oracle
   update cadence.

For LEZ, the realistic at-launch posture is (1) for primary, with (2) as
redundancy for liveness insurance. RFP-020 covers the SVM adapter port; running
additional relayer instances is a separate operational commitment that is
**not** required to use RedStone on LEZ in Pull mode.

### Pull vs Push: when does Push become worth the cost

Push trades persistent ecosystem-paid gas for cheaper per-consumer reads. The
break-even depends on read volume on each pair: if a feed is read N times per
heartbeat interval and the per-update gas is roughly equal to per-read calldata
gas, the crossover is around N=1 reads per heartbeat. In practice, dedicated
feeds for a single high-throughput lending protocol can justify Push at LEZ
scale; shared feeds consumed by occasional dapps typically cannot.

At chain launch with no TVL, Pull is the right default. Push becomes worth
revisiting once a specific feed sees concentrated, sustained read demand that
exceeds the gas budget of a typical relayer cadence.

## Signature Verification Schemes

The cost of verifying oracle signatures on-chain dictates whether LEZ can host
an oracle adaptor at all. LEZ is a RISC-V zkVM execution environment built on
RISC0. The on-chain signature primitive currently wired into the runtime is
single-key secp256k1 Schnorr (BIP-340) over SHA-256, validated as a witness on
the transaction (the runtime checks the signature when it admits the
transaction). This primitive is **not exposed to guest programs**: a program
running inside the RISC-V zkVM cannot invoke it as a host function. No threshold
or aggregate scheme (BLS, Schnorr multisig, t-Schnorr) is exposed to guest
programs either, and no other signing scheme has a host primitive at all. Any
signature that a program needs to verify (whether BIP-340 Schnorr from a DLC or
FROST publisher, secp256k1 ECDSA from RedStone or Pyth, ed25519 from
Switchboard) has to run as program code inside the RISC-V zkVM, where
verification cost is dominated by the ZK proving overhead of the underlying
primitive. ECDSA recovery and keccak256 are both expensive to prove; in-circuit
Schnorr/SHA-256 performance on RISC0 is currently unmeasured. The cost question
is identical in shape across signature schemes: how much does in-circuit
verification cost, and is that acceptable for the adaptor's update cadence.

This matters because every general-purpose price oracle in production today
(RedStone, Pyth via Wormhole, Chainlink Data Streams, Chronicle's per-signer
leg) signs with secp256k1 ECDSA over keccak256, and Switchboard signs with
ed25519. None of these match the LEZ-native primitive. Verifying their payloads
on chain therefore requires either (a) adding a secp256k1 ECDSA + keccak256
precompile to LEZ, or (b) running a trusted relayer that re-signs upstream
payloads in BIP-340 Schnorr over SHA-256, which collapses the trust set from N
publishers to one re-signer. This section walks through what schemes Pyth and
RedStone actually use, and what they cost on chains that do expose the matching
primitive, so the gap between the upstream cost profile and the LEZ-side cost is
visible to the reader.

### Wormhole VAA verification (Pyth dependency)

A Verifiable Action Approval (VAA) is the canonical Wormhole attestation. The
guardian set is a fixed roster of 19 nodes; a super-majority of 13 signatures is
required for a VAA to be considered valid [19][20]. Each guardian signs the
double-keccak256 hash of the VAA body (single keccak256 on Solana, because the
Solana secp256k1 program hashes the message itself) [19]. The signature scheme
is plain m-of-n ECDSA over secp256k1, with no aggregation: the VAA carries 13
independent (v, r, s) tuples plus a guardian index per signature [19][21].
Wormhole's own protocol documentation explains the choice: BLS aggregation would
cost roughly 130K gas to verify on Ethereum, whereas plain ECDSA via the
ecrecover precompile costs about 5K gas per signature, so the simpler scheme
wins on cost despite the linear verification work [21][22].

On EVM chains, the core bridge contract calls verifyVM, which invokes
verifySignatures and loops over the 13 signatures one by one, calling ecrecover
for each [21][22]. The ecrecover precompile has a fixed base cost of 3,000 gas
[23]; combined with calldata costs, the documented per-signature cost is
approximately 5K + 5K (recovery + calldata), giving roughly 130K gas for the
signature step alone before Merkle proof verification and storage writes [22].
On Solana, Wormhole calls the native secp256k1 precompile via verify_signatures,
but Solana's per-call compute-unit budget forces the verification to be split
across multiple instructions, each verifying six or seven signatures at a time
[22][24]. The Solana secp256k1 precompile costs approximately 6,690 compute
units per signature verified [24][25]. For Pyth specifically, the Perseus
upgrade amortises the VAA verification step across multiple price updates: a
single set of Wormhole signatures is verified once per transaction regardless of
how many feeds are updated, yielding a 50 to 80 per cent gas reduction when
updating five feeds at once [26].

Alternative verification paths exist but are not in production for Pyth.
Wormhole has shipped a Boundless ZK verifier (RISC Zero Groth16 proofs of
Ethereum consensus) deployed on Ethereum, Base, Optimism, Arbitrum, Linea, and
Avalanche, with a two-of-two policy that requires both a valid guardian
signature set and a valid ZK proof before a transfer executes [27]. The
wormhole-foundation/example-zk-light-clients repository contains ZK light client
templates for various source chains [27]. None of these supersede the 13-of-19
ECDSA path for Pyth price-feed delivery; they augment it.

### RedStone per-chain signatures

RedStone signs every data package with one scheme everywhere it operates: ECDSA
over secp256k1 with a keccak256 message hash, matching Ethereum's signing
convention [28][29]. The reference implementation lives in the
redstone-finance/rust-sdk crate; its Cargo.toml depends on either the secp256k1
crate (with the recovery feature) or the k256 ECDSA crate, plus sha3 for keccak
[29]. The crypto module exposes recover_public_key and recover_address functions
that validate signature malleability against the secp256k1 curve order and
accept Ethereum-style recovery bytes (0, 1, 27, 28) [29]. The on-chain EVM
consumer contract calls SignatureLib.recoverSignerAddress on
keccak256(signedMessage), then enforces a per-feed unique-signer threshold via
getUniqueSignersThreshold(); RedStone's documentation recommends at least three
unique signers as a balance between security and gas cost [28][30].

The signing scheme does not change per chain; only the verification primitive
does. Each chain-specific connector recovers the same secp256k1 ECDSA signatures
using whatever host-chain primitive is available:

| Chain             | Verification primitive                          | Source   |
| ----------------- | ----------------------------------------------- | -------- |
| EVM               | ECRECOVER precompile (secp256k1, keccak256)     | [28][30] |
| Solana            | secp256k1_recover syscall / secp256k1 program   | [25][29] |
| Sui               | sui::ecdsa_k1::secp256k1_ecrecover Move builtin | [31]     |
| Stellar (Soroban) | recover_key_ecdsa_secp256k1 host function       | [32][33] |
| Fuel              | Sway contract using secp256k1 recovery          | [34]     |
| Radix (Scrypto)   | Rust SDK with secp256k1 / k256 crate            | [29][35] |
| Casper            | Rust SDK with secp256k1 / k256 crate            | [29]     |

Note: although Stellar's native account model uses ed25519 (referenced in
RedStone's Stellar connector deployment documentation [36]), RedStone's
data-package signing and verification on Stellar use the same secp256k1 ECDSA
over keccak256 as on every other chain, via Soroban's
`recover_key_ecdsa_secp256k1` host function [32][33].

Per-chain verification cost (single signature):

| Chain             | Cost per ECDSA recovery                         | Source   |
| ----------------- | ----------------------------------------------- | -------- |
| Ethereum / EVM    | 3,000 gas (precompile) plus calldata            | [23]     |
| Solana            | approximately 6,690 compute units               | [24][25] |
| Stellar (Soroban) | approximately 2.3M CPU instructions (2,315,295) | [32]     |

For an M-of-N RedStone payload with the recommended threshold of three unique
signers [30], total verification cost on EVM is approximately 9,000 gas for the
precompile calls plus calldata and signer-bitmap accounting; the documented
end-to-end gas cost of a RedStone update on EVM falls in the 50K to 100K range
[3] (consistent with the table in External Oracle Models above).

### Implications for LEZ

The discussion below applies only to off-chain price oracles (external
publishers signing data that has to be verified on chain). The on-chain TWAP
tier is structurally separate: it reads LEZ-native AMM pool state, accumulates
price observations, and exposes them through a program account. No external
signature is involved, so the LEZ-native single-sig BIP-340 Schnorr primitive is
sufficient (it covers transaction authentication, not data attestation). RFP-019
sits entirely on the on-chain side and is unaffected by what follows.

For the off-chain side, the gap is real: every general-purpose price oracle in
production today signs with secp256k1 ECDSA over keccak256 (RedStone, Pyth via
Wormhole, Chainlink, Chronicle's per-signer leg) or ed25519 (Switchboard). None
match the LEZ-native primitive. The candidates that *do* sign in BIP-340 Schnorr
over tagged SHA-256 are concentrated in the Bitcoin DLC ecosystem (Pythia from
DLC Markets, Sibyls, Suredbits, Ernest Oracle on Nostr), all of which are
single-operator BTC/USD publishers built around discrete-event attestation
rather than continuous price streams. None today publish ZEC/USD or XMR/USD, and
none are decentralised in the way a DeFi-grade feed needs.

Four realistic adaptor shapes exist for closing this gap. They are independent
of RFP-019. A constraint that runs across C and D: no signature-verification
primitive is currently exposed to guest programs on LEZ, so any in-program
signature check (whether ECDSA-keccak for shape D's adaptor or BIP-340 for shape
C's DLC attestations) runs as RISC-V code inside the RISC0 zkVM. The cost
question is the same shape across both; only the upstream supply differs. Shapes
A and B avoid the question because the signature being checked authenticates the
LEZ transaction itself, not data carried inside calldata: the transaction sender
is a regular LEZ user (a single re-signer in A, a t-of-n FROST federation
emitting one aggregated BIP-340 signature in B), the runtime validates the
BIP-340 transaction witness at admission time as part of the standard
transaction-admission flow, and the price-aggregator program does only an
equality check on the authenticated caller against a registered pubkey. There is
no in-program signature verification under either shape, so the in-circuit cost
question never arises in public push mode. The remaining variable is trust:
collapse to one re-signer (A) or to t honest signers in a federation (B).
Crucially, neither shape unlocks private-execution pull: both are public
push-mode designs, with private consumers reading the public price account
written by the relayer or federation.

The structural test that distinguishes the shapes is **whose signature
authenticates the LEZ transaction**. If it is the transaction sender's (a single
re-signer in A, a FROST federation emitting one aggregated BIP-340 signature in
B), the runtime handles verification at admission and the guest program does an
authorisation check on the caller. If the LEZ transaction carries an upstream
publisher's signature inside calldata, distinct from the transaction sender, the
guest program has to verify it in-circuit (shapes C and D).

**Shape A — Trusted re-signer relayer.** A LEZ-side process fetches RedStone or
Pyth payloads, verifies them off chain, and submits a regular LEZ transaction
that calls the price-aggregator program with the resulting price. The relayer's
BIP-340 Schnorr signature is on the transaction itself; the runtime validates it
at admission time. The aggregator program checks the authenticated caller
against a registered relayer pubkey (an equality check, not a signature
verification) and writes the price to a public price account. The trust set
collapses from N upstream publishers to one re-signer; the chain has no
cryptographic evidence that the relayer reported what the publishers actually
signed.

**Shape B — FROST-BIP340 federation, transaction-signing.** A t-of-n federation
jointly signs the LEZ transaction itself (not the data inside calldata) using
FROST, emitting a single BIP-340-verifiable Schnorr signature as the transaction
witness. Each federation member fetches upstream RedStone or Pyth payloads
independently, verifies the upstream signatures off chain in native code, and
proposes a `(price, timestamp, source_metadata)` tuple. When t members agree on
the tuple within a tolerance window, they run FROST signing rounds (commit +
sign) over the LEZ transaction hash and the coordinator submits the resulting
tx. The aggregator program checks the authenticated caller against a registered
federation aggregate pubkey `P_fed` (an equality check, not a signature
verification) and writes the price to a public price account. The signing
infrastructure exists as libraries (Zcash Foundation FROST [45], Blockstream
`bip-frost-dkg` [46], Frostsnap [47]; jesseposner FROST-BIP340 [48] is the
reference implementation), and ZF is actively building FROST tooling for Zcash,
which aligns with the privacy-asset focus.

**Assumption to validate by PoC: the runtime accepts the** **federation's
FROST-aggregated BIP-340 signature as a normal LEZ** **transaction witness.**
FROST output is byte-identical to a single BIP-340 signature under an aggregate
public key, so the runtime's existing tx-admission BIP-340 primitive should
validate it without changes. If that assumption holds, the federation's write tx
pays only the standard tx-admission verification cost (host-program BIP-340
verification, runs outside the RISC-V zkVM circuit), and the price-aggregator
program does no in-program signature verification on the write side. The
compute-unit cost of pushing a price update is therefore lower than under shape
D, where the aggregator program verifies an upstream data signature in RISC-V
code at in-circuit cost. If the assumption fails (the runtime rejects the
FROST-aggregated witness for some reason, or operationally the federation cannot
be set up to sign LEZ transactions directly), the federation would have to fall
back to verifying a data signature in program code, raising the public-mode push
CU cost to the in-circuit Schnorr cost the bench measures (5:22 E2E private TX
vs 7:26 ECDSA at 3-of-N on consumer CPU,
[`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)).
The reasons such a fallback might be forced are an open question that the PoC
has to surface.

**Either way, shape B is push-only mode. It does not unlock**
**private-execution pull.** Pull mode for a private consumer means the
consumer's own private transaction authenticates the upstream data inside its
own ZK circuit. Shape B authenticates a *write* transaction submitted by the
federation; the data the private transaction reads is the public price account
written by the federation. Private-mode reads under shape B are identical to
push-mode reads under shape D: read the slot, no signature work in the read
path. Private pull (the consumer's private tx verifies an upstream signature
inline) remains foreclosed under shape B for the same reason it is under shape
D: the upstream signature still has to be verified somewhere, and putting it in
the consumer's privacy circuit is what the bench rules out.

No price-oracle product is deployed in this shape today. Public framing of FROST
by its implementers and grant funders is exclusively wallet and custody
(Blockstream `bip-frost-dkg` README, ZF FROST documentation, OpenSats and Brink
grants for jesseposner/FROST-BIP340 and Frostsnap, Blockchain Commons HRF 2025
FROST grant for shared-custody multisig). The closest production precedent is
iBTC Network (formerly dlcBTC; operator rebranded DLC.Link to BitSafe in 2025),
which runs a t-of-n attestor federation at sizes 10-of-15 (iBTC on EVM) and
7-of-10 (CBTC on Canton, mainnet October 2025). The federation runs two parallel
signing modes \[51\]: per-attestor secp256k1 ECDSA over keccak256 for EVM-bridge
attestations (verified on chain by `ECDSAUpgradeable.recover` per signature, not
aggregated), and a single FROST-aggregated BIP-340 Schnorr signature inside the
Taproot spend path on Bitcoin. The FROST-BIP-340 path is therefore live but only
inside a Bitcoin script; there is no off-chain wire format that a downstream LEZ
verifier could subscribe to as a single BIP-340 stream. The attestation content
is also contract outcomes (was a burn observed on the counterparty chain) rather
than continuous price data, and the FROST library used is the project's own
`DLC-link/conduition-frost` (a fork-of-fork of ZF FROST) rather than
Blockstream's `bip-frost-dkg` or jesseposner's implementation. Chainflip runs
FROST in production at 100-of-150 for cross-chain vault signing [52], showing
FROST scales operationally, but its use is internal transaction signing rather
than external attestation. Babylon EOTS [53] uses BIP-340 Schnorr but is
per-validator (not threshold) and signs consensus votes, not external data. The
only academic proposal specifically for FROST-as-oracle is *FrostOracle* (Chen
et al., IEEE iThings 2023, [54]), which describes the construction but has no
known implementation.

A consequence of FROST's round-stateful design is that nonce-management
discipline for repeated signing differs from the one-shot wallet-ceremony model
the existing libraries are scoped to. Public reference deployments of FROST for
high-frequency repeated signing (such as a heartbeat-driven price update) do not
exist, and the existing audits of ZF FROST and `bip-frost-dkg` cover the
wallet-custody threat model rather than an oracle-shaped one.

**Shape C — DLC-oracle extension.** A handful of DLC oracle publishers emit
BIP-340 attestations natively.

Two disqualifiers apply, either of which is sufficient on its own. First, shape
C carries the same in-circuit cost question as shape D: verifying a DLC
attestation requires the guest program to verify BIP-340 Schnorr in-circuit at
unmeasured cost, multiplied by N (the bit-precision of the numeric DLC
encoding). Pursuing shape C is therefore not the right call unless LEZ later
exposes Schnorr verification to guest programs at acceptable cost. Second, even
with cheap Schnorr verification, the structural fit of the DLC attestation model
is prediction markets and discrete-outcome contracts (which is what the format
was designed for), not streaming price feeds for DeFi protocols. Either
condition alone moves shape C out of scope for the current oracle work; the
description below documents the ecosystem state for reference and for a future
prediction-market RFP.

A DLC oracle pre-announces nonce points (R-values) for a future event with a
known maturity time, then at maturity publishes the s-values that, combined with
the pre-committed R-points, yield BIP-340 Schnorr signatures over a hash of the
outcome. The native cadence is "one attestation per scheduled event," which
matches "did BTC settle above $X on date D" but does not match "what is BTC/USD
right now, updated every 30 seconds." For a continuous price feed, every update
has to be modelled as a scheduled event in advance, which is an unusual usage
pattern relative to what existing publishers operate.

#### Two signing conventions in the live ecosystem

The dlcspecs `Oracle.md` text mandates a tagged SHA-256 construction with domain
`DLC/oracle/announcement/v0` for the announcement signature and
`DLC/oracle/attestation/v0` for the attestation signature. The live ecosystem
does not implement this uniformly. Two distinct conventions exist, both calling
themselves dlcspecs-compatible:

- **Plain SHA-256 lineage.** Pythia (DLC Markets) [49], the P2PDerivatives
  reference oracle [56], and the rust-dlc reference verifier all use plain
  `SHA256(message)` with no tag. Pythia inherited this from sibyls but removed
  sibyls' dual-mode support; rust-dlc's `OracleAnnouncement::validate` and
  `OracleAttestation::validate` follow the same plain-SHA-256 path. Anything
  verified against this lineage will not verify under a strict reading of
  `Oracle.md`.
- **Tagged SHA-256 lineage.** Kormir [57] (active reference library), Ernest
  Oracle [58] (which delegates to Kormir), and Sibyls in `dlc_v0` mode (the
  `SigningVersion` selected by Lava's shipped `config/oracle.json` before the
  operator wound down) use the tagged construction byte-for-byte per the spec.

The two lineages produce different signed bytes for the same underlying message.
A LEZ-side verifier consuming attestations from this ecosystem must either pick
a lineage and reject the other, or maintain both code paths. Choosing the
rust-dlc / plain-SHA-256 path captures the more numerous and more
historically-deployed publishers (Pythia is the only one of those currently
live); choosing the spec-correct tagged path captures Kormir, Ernest, and any
future deployments that follow Kormir's canonical reference.

#### State of the live publishers (May 2026)

- **Pythia (DLC Markets) \[49\]:** live mainnet,
  `https://pythia.dlcmarkets.com`, cron every minute, BTC/USD only, single
  oracle pubkey, no rotation, no public attestation index (consumers must
  already know the maturity timestamp). Plain SHA-256 lineage.
- **Sibyls (Lava) \[50\]:** operator dead. `oracle.lava.xyz` returns 404
  (Wayback last-alive 2025-04-14, dead by 2025-11-12). The `lava-xyz/sibyls`
  GitHub repo has been deleted; the codebase exists only on a third-party mirror
  (`briefgaming/sibyls`) whose owner is unaffiliated with Lava. Lava itself
  abandoned DLCs in late 2025 and went custodial. No surviving Sibyls operator.
- **P2PDerivatives oracle (Crypto Garage) \[56\]:** repo dormant since
  2022-05-24. Last historical operator URLs verified dead
  (`oracle.10101.finance`, `oracle.lava.xyz`). Library code in `rust-dlc`
  continues; the application code is frozen. The only multi-asset DLC oracle in
  the survey (BTC/USD plus BTC/JPY) but not currently published.
- **Kormir \[57\]:** active library, monthly releases continuing through March
  2026\. Live reference deployment at `kormir.dlcdevkit.com` is dev/test data
  only. Operator runbook is minimal (Postgres plus a Nostr nsec key); no bundled
  price feed or scheduler.
- **Ernest Oracle \[58\]:** on hiatus since 2025-06-02. The OpenSats blog
  characterises Ernest as a Nostr publisher; the daemon does not import Nostr
  in-tree and exposes only an HTTP API on port 3001 (the Nostr publication path
  lives in `kormir-server` which Ernest does not deploy). Implements four
  Bitcoin chain-metric attestations (hashrate, fee rate, block fees,
  difficulty); the UTXO-size metric mentioned in the announcement post is not in
  the source. No public deployment located. Repo is unlicensed (no LICENSE file,
  no `license` field in `Cargo.toml`).
- **Magnolia Financial price oracle \[59\]:** live commercial, powers Lygos
  institutional Bitcoin lending. Attestations are dlcspecs-shaped BIP-340 (per
  the operator's public statements) but not publicly retrievable: the documented
  endpoint `GET /oracle/events/{eventId}` requires an API key, and there is no
  public attestation explorer or relay. Closed source.
- **v0l on Nostr (kind 1009) \[60\]:** live single-publisher feed. The signature
  is BIP-340 native (Nostr's signature scheme) but the message is
  `serialised_event_json` per NIP-01 with plain SHA-256, not the dlcspecs
  construction. Cannot be reused as a DLC attestation without the publisher
  dual-signing. The successor proposal NIP-1658 defines kinds 31892 / 1892 /
  10041, not kind 1009; kind 1009 is informal and not in the official NIPs
  registry.

#### Numeric DLC verification cost on LEZ

The numeric DLC encoding signs the outcome bit-by-bit: a price attested with
N-bit precision requires N nonce-point announcements up front and N independent
BIP-340 signatures at maturity, which the consumer chain verifies in sequence.
For 18-bit base-2 precision (covering integer dollar amounts up to roughly
$262,000; cent granularity for prices in that range would require 27+ binary
digits, which is what Pythia's production config uses at 30 digits), that is 18
single-sig Schnorr verifications per update on LEZ. As noted in the Signature
Verification Schemes section, BIP-340 verification is not exposed to guest
programs on LEZ, so each of those 18 verifications runs in-circuit; the
per-update cost is therefore 18× whatever in-circuit BIP-340 + SHA-256
verification costs in RISC0 (currently unmeasured). If LEZ later exposes Schnorr
verification as a host primitive at low cost, the multiplier becomes a constant
overhead instead of dominating; until then, shape C inherits the same in-circuit
cost question that shape D's ECDSA path does.

#### Trust and decentralisation

Trust is single operator per oracle. DLC's multi-oracle pattern combines
independent attestations via t-of-t adaptor signatures on the Bitcoin spend path
[55]; the LEZ analogue is an aggregator program that registers K independent
BIP-340 publishers and requires M-of-K agreement within a tolerance window. The
publishers do not coordinate, no DKG is involved, and each publisher remains a
single-key DLC oracle. With Sibyls dead, P2PDerivatives dormant, Magnolia
closed, and Ernest in hiatus, the realistic candidate set for an LEZ M-of-K
federation today is one (Pythia) plus whatever forks an external builder stands
up.

#### Privacy-asset coverage

None of the live BIP-340 publishers attest XMR/USD or ZEC/USD. Pythia's roadmap
covers BTC options, not non-BTC pairs. Forking Pythia or Kormir for additional
asset pairs is a few hundred lines of code (per the deep-research notes for both
projects); the harder constraint is operator obligations (key custody, rotation,
uptime) and pricefeed selection (LN Markets and BitcoinAverage are BTC-only;
Kraken delisted XMR/USD for US users; coverage on Gate.io and Bitstamp is
patchy).

The cleaner long-term home for shape C is a future prediction- market RFP, where
the discrete-event attestation model is the native fit and the operational
pattern matches what DLC publishers already run.

**Shape D — secp256k1 ECDSA on LEZ.** Verify RedStone (or Pyth) secp256k1 ECDSA
\+ keccak256 signatures on the LEZ side. Two implementation paths share the same
adaptor program shape; only the verification call site differs.

**Path D1 (day one): RISC-V in-program verification.** Implement ECDSA recovery
and keccak256 hashing as program code running inside the RISC0 zkVM, using
existing Rust crates (k256 / sha3 / equivalents) proved by RISC0 along with the
rest of the program. This is what RFP-020 commits to for the public-mode write
side. No runtime change required. The cross-scheme bench
[`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)
establishes naive in-circuit ECDSA is slow on consumer CPU: end-to-end private
TX time (privacy wrap plus sequencer roundtrip) for ECDSA secp256k1 at 3-of-N is
**7:26** on a CPU-only AMD Ryzen 9 7940HS (16 threads, no CUDA, no Bonsai), and
no scheme in the four-way matrix (ECDSA secp256k1, Schnorr secp256k1, ECDSA
P-256, Ed25519) lands under 30 s. That rules out private-execution pull mode
under D1 absent a RISC0-specific signature-verification accelerator or GPU /
Bonsai proving. Public-mode cost remains the open variable RFP-020 measures. The
bench above is a proving-cost benchmark (private-execution path); the
public-mode aggregator does no proving, so its write-side cost is in LEZ runtime
compute units rather than proof time and is not captured by these numbers. Real
LEZ devnet measurement of the aggregator's compute-unit cost is what RFP-020
commits to. Public-mode cost amortises across all downstream reads, so a
write-side cost that would be unworkable per-private-transaction may still be
acceptable per heartbeat.

For reference, the bench's per-signature user-cycle deltas (N=1, sub-noop) and
end-to-end private-TX rankings at 3-of-N:

| Scheme            | user cycles / sig (N=1) | E2E private TX (3-of-N) |
| ----------------- | ----------------------: | ----------------------: |
| ECDSA P-256       |                   198 K |                    4:58 |
| Schnorr secp256k1 |                   271 K |                    5:22 |
| ECDSA secp256k1   |                   303 K |                    7:26 |
| Ed25519           |                   803 K |                   11:09 |

P-256 is roughly 32% cheaper per-sig than secp256k1 ECDSA in this stack (sha256
prehash vs keccak256 dominates the gap). Schnorr secp256k1 is roughly 9% cheaper
than ECDSA secp256k1 on the same precompile path. Ed25519 is the most expensive
of the four, despite curve25519-dalek's accelerated RISC0 backend, because
Edwards arithmetic plus the in-algorithm sha512 (no zkVM precompile) dominates.
These per-sig deltas matter for any future "private-mode-friendly upstream"
follow-on; they do not change the day-one D1 picture, which is gated by
private-execution pull being infeasible at every scheme on CPU.

Caveats: synthetic same-message fixtures, no batch-verify shortcuts, and the
bench is an AI-assisted research repository explicitly not intended for mainnet.
The numbers are order-of-magnitude indicators; production LEZ measurement is
RFP-020 Deliverable D1.

**Path D2 (cost-conditional follow-on): an accelerated precompile or host
function in public-execution mode.** Triggered only if D1 is too expensive at
the push-mode aggregator's production cadence. A precompile lives outside the ZK
proof boundary, so a public-mode program calls it as native code. The upstream
cost reference points are approximately 6,690 CU per recovery on Solana [24][25]
and an end-to-end RedStone EVM update in the 50K to 100K gas range [3]; LEZ
public-mode cost via a precompile would track the lower of these two on a
constant-overhead basis [22][29]. D2 is the optimisation path, not a
precondition.

#### Public-mode aggregator design (applies to both D1 and D2)

The adaptor runs in public execution: the verifier executes once per update on
the write side, and the verified price plus timestamp land in a public price
account. Private-execution programs compose by reading the public account, not
by carrying signed payloads inline. This is push mode. The reasoning for
preferring it differs slightly between D1 and D2, but the end-state design is
identical.

Under D1, in-program verification is technically reachable from private
execution (the same RISC-V code can run inside a user's private proof), but
bench data
([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench))
puts end-to-end private TX time at 7:26 for ECDSA secp256k1 at 3-of-N on a
CPU-only Ryzen 9 7940HS, with no scheme in the four-way matrix landing under 30
s. That makes private-execution pull mode infeasible in practice on consumer
CPU, not merely expensive, absent a RISC0-specific signature-verification
accelerator or GPU / Bonsai proving. Push mode amortises the cost once across
all downstream reads on the public-mode write side instead.

Under D2, the asymmetry becomes structural rather than economic. A precompile is
unreachable from private execution: anything in a private transaction has to be
expressible inside the RISC-V zkVM circuit, and a host function lives outside
it. Private execution that wants to verify a secp256k1 signature would have to
either (a) verify in the privacy circuit (forfeits batching, pays unmeasured
RISC0 EC cost, defeats the precompile's purpose), or (b) place the signature in
the transaction's journal and break privacy. Neither option preserves both
efficiency and privacy. So under D2 push mode is not a preference but the only
design that works.

In both cases, push mode is the right choice. Under D1 the amortisation argument
carries it; under D2 the structural argument forces it. The aggregator program
is the same either way, which is what makes D2 a localised swap-in for D1 if
cost forces the upgrade.

Pull-mode reads are therefore deliberately out of scope:

- **Pull mode from public execution under D1.** Technically possible (the
  in-program verifier can be called from a public consumer's transaction), but
  it pays full proving cost per consumer transaction with no amortisation, which
  is strictly worse than reading the push-mode aggregator's public price
  account. Excluded by design.
- **Pull mode from private execution under D1.** Possible at a heavy
  proving-cost penalty for every private transaction. Excluded by design.
- **Pull mode from public execution under D2.** Available (a public consumer can
  carry a signed payload and call the precompile inline), but offers no benefit
  over reading the push-mode aggregator's public price account. Out of scope.
- **Pull mode from private execution under D2.** Structurally unavailable
  (precompile cannot be called from inside the privacy circuit). Out of scope.

A clarification on the demand side. The framing above treats private-execution
pull mode as a desirable capability that the current paths foreclose; whether
any consumer protocol on LEZ actually needs it is a separate question, and not
one that has been settled. Some consuming protocols already have design reasons
to keep specific actions in public transactions: the LSC stablecoin in
[RFP-013](../RFPs/RFP-013-reflexive-stablecoin-protocol.md), for example, places
parts of its flow in public execution for constraints unrelated to oracle
access, so a private-execution pull path would not change how the stablecoin
reads prices for those actions. A future follow-on RFP that proposes either a
RISC0-friendly upstream signature scheme or any other route to private-execution
pull should therefore start by establishing that some downstream consumer
genuinely needs the capability, not just by selecting a primitive that admits
cheaper in-circuit verification.

#### Broader open issues with adding a secp256k1 primitive (path D2 only)

Beyond the oracle adaptor, the broader question of "what does a secp256k1
primitive in LEZ unlock" has open design issues that LEZ runtime developers have
flagged but not resolved. The Solana precompile's main published use case is
letting Ethereum users authorise transactions on Solana with their existing
Ethereum keys; mapping that flow to LEZ raises additional questions:

- **Nullifier tracking.** Replay protection for secp256k1 signatures used in
  either public or privacy execution requires tracking nullifiers so the same
  signature cannot be reused.
- **Privacy-circuit branching.** Supporting private accounts authorised by an
  Ethereum signature (rather than the LEZ-native `nsk`) requires branching logic
  in the privacy circuit.
- **Account-identifier flow.** A recent LEZ change introduces identifiers that
  let one private-key set support multiple private accounts, with maintenance
  and recovery encrypted under the viewing public key. Identifiers are not
  currently supported for public accounts (each public account requires a fresh
  key set), so an Ethereum-signed public-account flow would imply fresh Ethereum
  accounts per use, and Ethereum-signed private accounts would require
  LEZ-specific wallet support to handle the identifier flow.

These issues are not blockers for the narrow oracle-adaptor use of the
precompile (push-mode aggregator writing a public price account), but they are
part of why the LEZ runtime team is not currently championing a precompile
addition: the cost is real, the in-circuit elliptic-curve performance is
unmeasured, and the compelling-use-case story beyond oracle adaptors is not yet
established. If RFP-020's cost measurement triggers a follow-on RFP for path D2,
that follow-on should be scoped on the assumption that the precompile is bespoke
runtime work that has to be argued for, not a small extension that is already on
the LEZ roadmap.

For private-account composability with off-chain price data, push mode is
therefore the structural design under both D1 and D2. The pull-vs-push analysis
below frames the same point in terms of the upstream RedStone / Pyth dichotomy.

#### Push mode is preferable to pull mode on LEZ

RedStone supports both pull (signed payload attached to consumer calldata) and
push (a relayer submits signed updates to an aggregator contract on a heartbeat
or deviation trigger). On LEZ the push model is the better fit:

- **Verification cost is paid once per update, not once per read.** The
  aggregator program recovers signatures and checks the unique-signer threshold
  on write, then stores the latest price and timestamp in a public account.
  Consumers just read the slot. The single write-side cost amortises across all
  downstream reads instead of being paid by every consuming transaction.
- **Composes cleanly with private accounts.** A private account reads a public
  price-feed account's slot the same way it reads any other shared public state.
  Pull mode is the awkward case: the consumer's transaction must carry the
  signed payload in calldata, coupling oracle data into the private execution
  path and making both signature recovery and payload handling part of the
  private workload. Under D1 this is technically reachable but pays the full
  in-circuit ECDSA cost in every private consumer's proof; under D2 it is
  structurally unavailable because the precompile cannot be called from inside
  the privacy circuit. Either way push mode is strictly better for private
  consumers.
- **Update cadence is a tunable parameter.** Heartbeat plus deviation threshold
  trade cost against freshness. For a TWAP oracle this is acceptable because the
  consumer is already smoothing; pull mode's "fresh at transaction time"
  guarantee is not required.

Tradeoffs: push mode requires someone to operate the relayer (RedStone runs the
pusher for their existing push deployments; a sovereign LEZ deployment would
rely on RedStone's relayer or run its own), and the aggregator program still has
to perform secp256k1 recovery and keccak256 hashing on the write side. Under D1
that path is RISC-V program code; under D2 it is a precompile call. The
write-side cost per update differs between the two but the design shape does
not.

There is one freshness pattern that is unusual to LEZ and worth calling out: a
user who needs a price fresher than the heartbeat's last update can submit a
**public transaction that pushes a fresh signed payload to the aggregator
account**, and then submit a **private transaction immediately afterwards that
reads the just-updated public price**. The verification cost is paid in the
public transaction (where it is cheaper, especially under D2 where the
precompile is callable), and the private transaction does no signature work at
all, just reads a slot. This pattern is uninteresting on chains without a public
/ private execution split because it collapses to ordinary pull mode, but on LEZ
it captures pull mode's "fresh at transaction time" property without paying the
in-circuit cost in the privacy proof. Cost is borne by the consumer, in the
public path, once per private action that needs a guaranteed-fresh price. The
aggregator program already accommodates this: any caller can submit a valid
signed payload and the program writes if signatures and timestamps check out.

A useful corollary: this enables a **consumer-pays push variant** that does not
require a dedicated relayer at all. Users push when they need a fresh price,
paying the verification cost themselves in the public path; idle periods incur
zero update cost; the aggregator only advances when someone actually needs it.
This is operationally pull (consumer-pays, on-demand) but structurally push (the
program owns the public price account that downstream private consumers read
from). Whether to run a heartbeat relayer in addition is a deployment-time
choice: a heartbeat keeps the slot warm for protocols that read it without first
pushing themselves; consumer-pays push keeps the cost model strictly
proportional to demand. Both can coexist; the program logic does not distinguish
between them.

#### How this differs from "put the signature in the journal"

The two-transaction split (public push, then private read) is not the same as
embedding the upstream signature in the private transaction's journal. The
differences matter:

- **Two distinct transactions, not one.** The signature is carried only in the
  public push transaction. The private transaction reads the resulting public
  price account by address, with no upstream signature in its calldata or
  journal. The private transaction's *contents* (which assets, which
  counterparty, which amount) remain private; only the fact that some price
  update happened is observable, and that fact is observable for any push
  regardless of who submitted it.
- **Fits the existing aggregator design.** The aggregator program already
  accepts signed payloads from any caller and writes a public price account;
  that is its normal write path. Consumer-pays push exercises this path from an
  end-user wallet rather than from a dedicated relayer. No new program logic.
- **Linkability risk to clarify, not to wave away.** An observer can correlate
  "wallet X pushes a price update at time T" with "private transaction at time T
  plus epsilon" and infer that the same actor is consuming the just-pushed
  price. Strength of the inference depends on push frequency and the consumer's
  wallet hygiene. Mitigations are consumer-side, not adaptor-side: push from a
  separate funding wallet from the one used in the private transaction,
  time-shift the push and consume across enough blocks that timing correlation
  weakens, or rely on a heartbeat relayer so that single-purpose pushes are not
  the observable pattern. None of these are guaranteed by the aggregator
  program; they are choices the consumer protocol or end user makes.

The privacy story is therefore strictly better than journal-disclosed signatures
(the private transaction's body stays private) but not equivalent to a
heartbeat-only push model (timing-correlation linkability remains). Whether the
residual linkability is acceptable is a consumer-side production-security
decision, the same way any DEX-routing or liquidity-management decision is.

RedStone is the simpler day-one option for the upstream-source side because it
carries no bridge dependency. Pyth depends on the full 13-of-19 VAA verification
plus Merkle proof verification, though it amortises across many feeds per
transaction after the Perseus upgrade [26]. Neither requires threshold or
aggregate schemes (BLS, Schnorr) on the *upstream* side, so the crypto surface
required by shape D is limited to secp256k1 ECDSA recovery with keccak256
hashing — implemented in-program under D1 (no runtime change) and exposed as a
precompile under D2 (cost- conditional follow-on).

## Production Oracle Architectures

Major lending and borrowing protocols have converged on multi-source oracle
designs with fallback mechanisms. These production patterns inform the
requirements and design rationale of RFP-019.

### Aave V3

Aave V3's `AaveOracle` contract uses Chainlink aggregators as the primary price
source via `getAssetPrice()`. Each asset is mapped to a Chainlink feed through
`setAssetSources()`. If the Chainlink feed returns a price \<= 0, the call is
forwarded to a configurable fallback oracle via `getFallbackOracle()` [15]. On
Layer 2 deployments, the `PriceOracleSentinel` contract monitors sequencer
uptime: if the L2 sequencer goes down, borrowing is disabled and liquidations
are paused for a configurable grace period (`setGracePeriod()`), giving users
time to restore position health after an outage [15]. Aave does not use TWAP as
a primary or secondary price source; it relies entirely on Chainlink feeds with
governance-managed fallback.

### Compound V2 and V3

Compound V2's `UniswapAnchoredView` contract implemented a two-source design:
Coinbase as the primary reporter, anchored against a Uniswap V2 TWAP. If the
Coinbase price diverged beyond 20% of the TWAP anchor, the price was rejected
and the system retained the last valid price [16]. This is the closest
production precedent to RFP-019's circuit breaker design. On 26 November 2020, a
DAI price spike to $1.30 on Coinbase triggered approximately $89M in
liquidations; the TWAP anchor limited the damage by rejecting the most extreme
prices, but the 20% tolerance was too wide to prevent all mispricing [16].
Compound V3 (Comet) dropped the TWAP anchor entirely and uses Chainlink feeds
directly via `AggregatorV3Interface`, with only zero-value validation; some
markets use derived-asset price wrappers (e.g. wstETH/ETH multiplied by ETH/USD)
but these are conversion adaptors, not bounds-checking circuit breakers [16].

### MakerDAO / Sky

MakerDAO's oracle pipeline has two layers: the Median contract (aggregates
prices from multiple whitelisted feed providers) and the Oracle Security Module
(OSM), which imposes a mandatory one-hour delay before new prices take effect in
the system [15a]. The delay is a circuit breaker by design: it gives governance
one hour to detect and respond to oracle manipulation before the system acts on
a compromised price. Emergency responses include calling `stop()` to freeze the
OSM or triggering Emergency Shutdown. MakerDAO does not use Uniswap TWAP.
Chronicle Protocol evolved from MakerDAO's internal oracle infrastructure, using
Schnorr signature aggregation to consolidate validator signatures into a single
"super signature," achieving constant-time verification at approximately 52K gas
regardless of the number of validators [15b].

### Liquity V2

Liquity V2 explicitly rejected Uniswap V3 TWAP as a primary or fallback source
due to liquidity migration risk (uncertainty about whether v3 liquidity would
persist after the Uniswap v4 launch) [18]. Instead, Liquity V2 uses Chainlink as
the primary oracle (composing multiple Chainlink feeds for LST collateral, e.g.
ETH/USD combined with an LST market-rate feed) with a simple fallback: if the
primary feed is frozen for more than 12 hours or returns bad data, the system
falls back to a secondary oracle (once, with no cascading). Liquity V2
explicitly prefers this simple fallback over complex circuit breaker designs
(e.g. Gyroscope's pause-on-divergence model), reasoning that pausing operations
requires human intervention to set a new oracle and unpause, which conflicts
with their immutability goals [18]. The design prioritises automation: simple
trigger conditions, single fallback, no manual intervention required.

### Common patterns

| Protocol    | Primary             | Secondary / fallback      | Circuit breaker                    | TWAP role             |
| ----------- | ------------------- | ------------------------- | ---------------------------------- | --------------------- |
| Aave V3     | Chainlink           | Governance-set fallback   | PriceOracleSentinel (L2 sequencer) | None                  |
| Compound V2 | Coinbase reporter   | Last valid price          | 20% TWAP anchor tolerance          | Anchor / sanity check |
| Compound V3 | Chainlink           | None (governance pauses)  | Zero-value check only              | None                  |
| MakerDAO    | Median (multi-feed) | Emergency Shutdown        | 1-hour OSM delay                   | None                  |
| Liquity V2  | Chainlink           | Simple fallback (one hop) | None (by design)                   | Rejected              |

Two patterns emerge. First, most major lending protocols use at least two tiers
of price validation (Aave, MakerDAO, Liquity V2, Compound V2): cross-check,
delay, or fallback. Compound V3 is the notable exception, relying on Chainlink
with only zero-value validation and governance-managed pause as its safety net.
Second, TWAP serves as an anchor or sanity check in the protocols that use it
(Compound V2), never as the sole price source for lending. These patterns
motivate RFP-019's two-tier architecture (on-chain TWAP + external feeds) and
circuit breaker design as a defence-in-depth choice rather than a universal
industry default.

## Oracle Service Levels and Feed Update Parameters

RFP-020 requires the awarded team to operate the day-one feeds under an SLA, so
the target numbers need grounding in what the incumbent oracle networks actually
commit to and how they parameterise feed updates. The headline finding is that
neither leading network publishes a numeric uptime guarantee for its public
feeds: both push outage risk onto the integrator and describe reliability
qualitatively rather than contractually.

### Neither Chainlink nor RedStone publishes a numeric uptime SLA

Chainlink states no uptime percentage for its public Data Feeds. Its
documentation instead makes reliability the integrator's responsibility,
advising developers to "include monitoring and safeguards to protect against ...
potential delays, and outages" [66]. Reliability is described in narrative terms
(multi-year track record, decentralisation at the data-source, node, and network
levels), never as a figure [66][67].

RedStone goes further and explicitly disclaims availability in its Terms of Use:
the service is provided "ON AN AS-IS AND AS-AVAILABLE BASIS," RedStone "cannot
guarantee the Site will be available at all times," and it accepts no liability
"for any loss, damage, or inconvenience caused by your inability to access or
use the Site during any downtime or discontinuance" [68]. The public price-feeds
page advertises reliability only qualitatively ("0 Mispricing Events," "trusted
by 100+ protocols across 80+ chains") with no availability percentage [69].

The consequence for RFP-020 is direct: any uptime target the awarded team
commits to is a target on its own operated relayer and monitoring stack, not a
figure inherited from RedStone's public gateways. RedStone's gateways carry no
contractual SLA to pass through.

### The one published figure we could locate is 99.9%

Absent a vendor SLA, the only concrete uptime figure we could find published for
a production oracle feed is 99.9% (roughly 43 minutes of downtime budget per
month): a report of Pyth's pull oracle maintaining 99.9% uptime during a period
of Solana congestion in which the push oracle missed a significant number of
updates [70]. That source is a single secondary article and is directional
rather than authoritative, but it is the closest thing to a published benchmark
in the survey. We found no oracle network publicly committing to a figure above
99.9% for its feeds, so a target of 99.99% or higher would rest on no published
precedent; 99.9% is the defensible ceiling to anchor on.

### Feed update parameters: deviation threshold and heartbeat

Both networks update a feed on a first-to-fire pair of conditions: a **deviation
threshold** (update when the off-chain price moves more than X% since the last
on-chain update) and a **heartbeat** (update when no update has occurred for Y
seconds regardless of deviation).

- **Chainlink.** The mechanism is documented, but per-feed values live in the
  dashboard rather than the prose docs [66]. A representative major crypto/USD
  feed (ETH/USD on Ethereum mainnet) uses a **0.5% deviation threshold** and a
  **3600-second (1-hour) heartbeat** [71]. Lower-cap or stable assets commonly
  use wider bands (1% to 2% deviation, up to 24-hour heartbeat).
- **RedStone (Push).** The relayer exposes the same two conditions as
  configuration: `UPDATE_PRICE_INTERVAL` (heartbeat) and
  `MIN_DEVIATION_PERCENTAGE` (deviation), and updates "if any conditions are
  met" [64]. RedStone deliberately does **not** publish fixed default values;
  the parameters are set per integrating protocol. So the operator running
  RedStone push on LEZ chooses these values to match the asset, and can match
  Chainlink's representative bands directly.

For LEZ's day-one asset list, the grounded parameterisation is therefore a 0.5%
deviation threshold and 1-hour heartbeat for the volatile crypto/USD pairs (BTC,
ETH, SOL, XMR, ZEC against USD), widening the band only if a specific asset
proves too thin or too volatile to sustain that cadence economically.

### Grounded servicing targets for RFP-020

Combining the above, the following targets are defensible (each is either the
closest published reliability figure the survey could locate or a documented
incumbent parameter, and none contradicts a published vendor claim):

| Parameter          | Grounded target                                  | Basis                                                                         |
| ------------------ | ------------------------------------------------ | ----------------------------------------------------------------------------- |
| Feed availability  | 99.9% monthly (~43 min/month downtime budget)    | Closest published figure located (Pyth 99.9% example) [70]; none found higher |
| Heartbeat          | 1 hour (volatile crypto/USD); up to 24h (stable) | Chainlink ETH/USD 3600s [71]; RedStone push configurable [64]                 |
| Deviation          | 0.5% (major pairs); 1% to 2% (less liquid)       | Chainlink ETH/USD 0.5% [71]; RedStone `MIN_DEVIATION_PERCENTAGE` [64]         |
| Incident detection | within one heartbeat interval                    | Operator-set; both networks push staleness detection onto the integrator [66] |
| Incident response  | acknowledge ≤ 1h, mitigate ≤ 4h                  | Operator-set; no vendor publishes a response-time SLA                         |

Two caveats carry into the RFP. First, because no vendor guarantees uptime, the
target must be backed by the awarded team's own operated relayer, monitoring,
and alerting, and paired with mandatory staleness checks and circuit breakers on
the consumer side (both Chainlink and RedStone push outage handling onto the
integrator [66][68]). Second, the availability figure should not be claimed
above 99.9% absent a real precedent that supports it.

## Privacy-Asset Feed Availability

LEZ's privacy focus likely makes XMR/USD and ZEC/USD first-class pricing
requirements. On-chain TWAP cannot supply these prices: no wrapped XMR or ZEC
token has sufficient DEX liquidity (low five to six figures across all surveyed
pools on Ethereum and Solana) for a manipulation-resistant TWAP, so an off-chain
feed is the only viable path on day one. Coverage across the surveyed oracles is
summarised below.

| Oracle                 | XMR/USD                                                                             | ZEC/USD                                                   | Self-serve on LEZ?                                                                                                                                                      |
| ---------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Chainlink (push)       | Active on Optimism (1,200s heartbeat, 0.2% deviation) and Polygon (24h, 1%) [39]    | Active on Ethereum (24h, 2%) and Polygon (24h, 1%) [40]   | No: permissioned onboarding                                                                                                                                             |
| Chainlink Data Streams | 43+ chains, subscription-gated [39]                                                 | 35+ chains, subscription-gated [40]                       | No: paid product                                                                                                                                                        |
| Pyth (pull)            | `crypto-xmr-usd`, multiple publishers across both Pyth clusters; live count at [41] | `crypto-zec-usd`, multiple publishers; live count at [41] | Yes, once Wormhole endpoint and the Pyth receiver are deployed                                                                                                          |
| RedStone (pull)        | Listed; data feed ID `XMR` [42]                                                     | Listed; data feed ID `ZEC` [42]                           | Yes once shape D1 or D2 lands per RFP-020 (LEZ-side ECDSA + keccak verification path); no bridge dependency on the upstream side, no per-chain RedStone team engagement |
| DIA / Lumina           | Production-ready (MAIR aggregation, 120s, announced January 2026) [43]              | Available (MAIR, 120s) [43]                               | Yes via Lumina; bespoke deployment per chain                                                                                                                            |
| Supra                  | XMR_USDT, 195 sources (Standard tier) [44]                                          | ZEC_USDT, 60 sources (Premium tier) [44]                  | No: requires Supra team engagement                                                                                                                                      |
| Chronicle              | Not in public feed catalogue                                                        | Not in public feed catalogue                              | N/A                                                                                                                                                                     |
| API3                   | Not found                                                                           | Not found                                                 | N/A                                                                                                                                                                     |
| Switchboard            | Not found                                                                           | Not found                                                 | Permissionless feed creation supported but no XMR or ZEC feed currently exists                                                                                          |

## LEZ Bootstrap Strategy

### Phase 1: Genesis (no TVL)

Deploy RedStone pull oracle contract (no bridge dependency) and, if Wormhole is
available, the Pyth EVM receiver contract. This provides 400 to 1,500+ price
feeds immediately with zero per-chain oracle node infrastructure [3][5].
RedStone is the practical day-one oracle because it requires no cross-chain
bridge; Pyth is conditional on Wormhole integration [8].

### Phase 2: Early growth (first DEX, some TVL)

Once RFP-004 (DEX) is live, deploy the on-chain TWAP oracle as a supplementary
data source. At this stage, TWAP should be used only as a sanity check (circuit
breaker comparison against external feeds), not as a primary price source,
because pool liquidity will be insufficient for manipulation resistance [10].

Engage Switchboard for core EVM contract deployment (TEE security,
permissionless custom feeds). Consider DIA Lumina as a permissionless fallback
for assets not covered by Pyth or RedStone [7].

### Phase 3: Maturity (significant TVL)

As DEX pools grow beyond $50M to $100M in depth, on-chain TWAP becomes a viable
secondary price source for lending and liquidation use cases. At this stage,
multi-source aggregation (median of TWAP + Pyth + RedStone) provides the highest
reliability [6].

Engage Chainlink for official DON deployment once LEZ meets infrastructure
requirements (3+ independent RPC providers, sufficient TVL, dedicated node
operators). Treat Chainlink as a 12 to 24 month post-launch milestone [4][8].

### Graduation path

The TWAP tier's role evolves with liquidity:

| Pool depth   | TWAP role                                  | Recommended window |
| ------------ | ------------------------------------------ | ------------------ |
| < $1M        | Not usable (manipulation trivially cheap)  | N/A                |
| $1M to $10M  | Sanity check only (circuit breaker)        | 2 to 12 h          |
| $10M to $50M | Secondary source (median with external)    | 30 min to 2 h      |
| > $50M       | Co-primary source alongside external feeds | 30 min to 1 h      |

## References

01. DefiLlama, "Oracles" dashboard, accessed Q1 2026.
    https://defillama.com/oracles
02. Mitosis University, "Which Oracle Powers What," 2025.
    https://university.mitosis.org/chainlink-pyth-redstone-chronicle-supra-switchboard-which-oracle-powers-what/
03. RedStone, "Blockchain Oracles Comparison: Chainlink vs Pyth vs RedStone
    2025," Jan 2025.
    https://blog.redstone.finance/2025/01/16/blockchain-oracles-comparison-chainlink-vs-pyth-vs-redstone-2025/
04. Chainlink, "Network Integration Requirements."
    https://docs.chain.link/resources/network-integration
05. Bai et al., "Ormer: A Manipulation-Resistant and Gas-Efficient Blockchain
    Oracle Scheme," arXiv:2410.07893v2, Oct 2024.
    https://arxiv.org/html/2410.07893v2
06. ChainSecurity, "Oracle Manipulation After The Merge," 2022.
    https://www.chainsecurity.com/blog/oracle-manipulation-after-merge
07. DIA, "Lumina" documentation, accessed 2026. https://www.diadata.org/lumina/
08. Pyth Network, "Cross-Chain Delivery" documentation.
    https://docs.pyth.network/price-feeds/core/how-pyth-works/cross-chain
09. Uniswap, "Oracle" (v3 protocol concepts).
    https://docs.uniswap.org/concepts/protocol/oracle
10. Uniswap Labs, "Uniswap v3 Oracles" (PoS analysis), 2022.
    https://blog.uniswap.org/uniswap-v3-oracles
11. SmartContent, "TWAP Oracles vs Chainlink Price Feeds: A Comparative
    Analysis."
    https://smartcontentpublication.medium.com/twap-oracles-vs-chainlink-price-feeds-a-comparative-analysis-8155a3483cbd
12. Chainlink, "Off-Chain Reporting" documentation.
    https://docs.chain.link/architecture-overview/off-chain-reporting
13. Switchboard, "Protocol" documentation.
    https://docs.switchboard.xyz/how-it-works/switchboard-protocol
14. Uniswap, "Oracles" (v2 protocol concepts).
    https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles
15. Aave, "Oracles" (v3 smart contracts documentation).
    https://aave.com/docs/aave-v3/smart-contracts/oracles 15a. MakerDAO, "Oracle
    Security Module (OSM)" documentation.
    https://docs.makerdao.com/smart-contract-modules/oracle-module/oracle-security-module-osm-detailed-documentation
    15b. Chronicle Labs, "Understanding Chronicle" documentation.
    https://docs.chroniclelabs.org/understandingChronicle
16. Compound Finance, "Price Feed" documentation.
    https://compound.finance/docs/prices
17. RedStone, "Blockchain Oracles Comparison: Chainlink vs Pyth vs RedStone
    2026," Mar 2026.
    https://blog.redstone.finance/2026/03/30/blockchain-oracles-comparison-chainlink-vs-pyth-vs-redstone-2026/
18. Liquity, "The Oracle Conundrum," 2023.
    https://www.liquity.org/blog/the-oracle-conundrum
19. Wormhole, "VAAs (Verifiable Action Approvals)" documentation, accessed Apr
    2026\. https://wormhole.com/docs/protocol/infrastructure/vaas/
20. Wormhole, "Guardians" documentation, accessed Apr 2026.
    https://wormhole.com/docs/protocol/infrastructure/guardians/
21. Wormhole, "Security" documentation, accessed Apr 2026.
    https://wormhole.com/docs/protocol/security/
22. Sec3, "How Do Cross-Chain Bridges Work? A Case on Wormhole (Part 2)," 2022.
    https://www.sec3.dev/blog/bridges2
23. Ethereum, "Precompiled Contracts" (ECRECOVER at address 0x01, base cost
    3,000 gas). https://www.evm.codes/precompiled
24. Solana Labs, "block_cost_limits.rs" (secp256k1 verification compute units).
    https://github.com/solana-labs/solana/blob/master/cost-model/src/block_cost_limits.rs
25. Solana Foundation, "Fee Structure" (Ed25519, Secp256k1, Secp256r1 precompile
    programs). https://solana.com/docs/core/fees/fee-structure
26. Pyth Network, "Perseus Network Upgrade" blog post, 2025.
    https://www.pyth.network/blog/perseus-network-upgrade
27. Wormhole, "Boundless Partners with Wormhole to Launch ZK Network Powered by
    RISC Zero," 2025.
    https://wormhole.com/blog/boundless-partners-with-wormhole-to-launch-zk-network-powered-by-risc-zero
28. RedStone, "How Data Flows to the Blockchain" architecture documentation,
    accessed Apr 2026. https://docs.redstone.finance/docs/architecture/
29. RedStone, "rust-sdk" repository (crates/redstone, including crypto/mod.rs
    and Cargo.toml dependencies on secp256k1 and k256).
    https://github.com/redstone-finance/rust-sdk
30. RedStone, "redstone-oracles-monorepo" EVM connector
    (RedstoneConsumerBase.sol, getUniqueSignersThreshold,
    SignatureLib.recoverSignerAddress).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/evm-connector/contracts/core/RedstoneConsumerBase.sol
31. Sui Documentation, "Module sui::ecdsa_k1" (secp256k1_ecrecover Move
    builtin). https://docs.sui.io/references/framework/sui_sui/ecdsa_k1
32. Stellar, `rs-soroban-env` budget metering calibration table
    (`RecoverEcdsaSecp256k1Key` const cost 2,315,295 CPU instructions). CAP-0051
    ("Smart Contract Host Functionality") defines the host function; the
    calibrated cost is in the `rs-soroban-env` test fixture below.
    https://github.com/stellar/rs-soroban-env/blob/main/soroban-env-host/src/test/budget_metering.rs
    https://github.com/stellar/stellar-protocol/blob/master/core/cap-0051.md
33. Veridise, "RedStone Stellar Connector" security assessment, Oct 2025.
    https://veridise.com/wp-content/uploads/2025/10/VAR-Redstone-251006-Oracles-SDK-V2.pdf
34. RedStone, "redstone-oracles-monorepo" Fuel connector (Sway contract
    adapter).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/fuel-connector/README.md
35. RedStone, "RedStone Brings Secure, Gas-Efficient Oracle Solutions to Radix
    DeFi Ecosystem," Jun 2025.
    https://blog.redstone.finance/2025/06/12/redstone-brings-secure-gas-efficient-oracle-solutions-to-radix-defi-ecosystem/
36. RedStone, "redstone-oracles-monorepo" Stellar connector DeployingFeed.md
    (Stellar account key uses Ed25519-curve; refers to deployer key, not
    RedStone signing scheme).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/570006fccf0f919ad9722d11914dd0bc1c5b136d/packages/stellar-connector/DeployingFeed.md
37. CoinDesk, "Bitcoin Flash Crashed to $5K on Pyth Network's Data Feed," Sep
    2021\.
    https://www.coindesk.com/markets/2021/09/22/bitcoin-flash-crashes-to-5k-on-pyth-networks-data-feed
38. Morpho governance forum, "PYTH CBETH price feed is easily manipulated,
    resulted in me losing $33,000," Mar 2025.
    https://forum.morpho.org/t/pyth-cbeth-price-feed-is-easily-manipulated-resulted-in-me-losing-33000/1577
39. Chainlink, XMR/USD price feeds: Optimism Mainnet
    (`0x2a8D91686A048E98e6CCF1A89E82f40D14312672`) and Polygon Mainnet
    (`0xBE6FB0AB6302B693368D0E9001fAF77ecc6571db`); Data Streams
    XMR/USD-RefPrice product on 43+ chains.
    https://data.chain.link/feeds/optimism/mainnet/xmr-usd
    https://data.chain.link/feeds/polygon/mainnet/xmr-usd
    https://data.chain.link/streams/xmr-usd-cexprice-streams
40. Chainlink, ZEC/USD price feeds: Ethereum Mainnet
    (`0x3f929667bdf783b99274F10465a89d6aF772736E`) and Polygon Mainnet
    (`0xBC08c639e579a391C4228F20d0C29d0690092DF0`); Data Streams
    ZEC/USD-RefPrice product on 35+ chains.
    https://data.chain.link/ethereum/mainnet/crypto-usd/zec-usd
    https://data.chain.link/feeds/polygon/mainnet/zec-usd
    https://data.chain.link/streams/zec-usd-cexprice-streams
41. Pyth Network, legacy price feeds dashboard (`crypto-xmr-usd`,
    `crypto-zec-usd`).
    https://insights.pyth.network/legacy-price-feeds/crypto-xmr-usd
    https://insights.pyth.network/legacy-price-feeds/crypto-zec-usd
42. RedStone, "ALL_SUPPORTED_TOKENS" registry (XMR and ZEC listed).
    https://github.com/redstone-finance/redstone-api/blob/main/docs/ALL_SUPPORTED_TOKENS.md
43. DIA, asset price index (XMR and ZEC).
    https://www.diadata.org/app/price/asset/Monero/0x0000000000000000000000000000000000000000/
    https://www.diadata.org/app/price/asset/Zcash/0x0000000000000000000000000000000000000000/
44. Supra, "Data Feeds Index" (XMR_USDT, ZEC_USDT).
    https://docs.supra.com/oracles/data-feeds/data-feeds-index
45. Zcash Foundation, FROST (Flexible Round-Optimised Schnorr Threshold)
    reference implementation and documentation. https://frost.zfnd.org/
46. Blockstream Research, `bip-frost-dkg` (FROST distributed key generation,
    BIP-340 compatible). https://github.com/BlockstreamResearch/bip-frost-dkg
47. Frostsnap, hardware-wallet stack using secp256kfun for FROST threshold
    Schnorr. https://github.com/frostsnap/frostsnap
48. Jesse Posner, `FROST-BIP340` (reference implementation of FROST emitting
    BIP-340-verifiable signatures). https://github.com/jesseposner/FROST-BIP340
49. DLC Markets, "DLC Markets open-sources its oracle Pythia," May 2025; Pythia
    source.
    https://blog.dlcmarkets.com/dlc-markets-open-sources-its-oracle-pythia/
    https://github.com/dlc-markets/pythia
50. Lava, `sibyls` (DLC oracle implementing BIP-340 attestation over numeric
    outcomes). URL no longer resolves as of 2026-05; see body discussion at
    "State of the live publishers" for operator and repo status. Third-party
    mirror at `briefgaming/sibyls` (unaffiliated).
    https://github.com/lava-xyz/sibyls
51. iBTC Network (formerly DLC.Link / dlcBTC), technical stack documentation
    describing the attestor federation (10-of-15 on EVM, 7-of-10 on Canton).
    Companion FROST blog post URL no longer resolves as of 2026-05; tech-stack
    page remains live. https://docs.dlc.link/tech-stack
    https://www.ibtc.network/blog/frost-at-dlc-link-pioneering-advanced-security-for-dlcs
52. Chainflip, "FROST Signature Scheme" protocol documentation (100-of-150
    threshold for cross-chain vault signing).
    https://docs.chainflip.io/protocol/frost-signature-scheme
53. Babylon Labs, "EOTS Manager" architecture documentation (per-validator
    BIP-340 Schnorr finality voting; not threshold).
    https://docs.babylonlabs.io/guides/architecture/btc_staking_program/eots_manager/
54. Chen et al., "FrostOracle: A Novel and Efficient Blockchain Oracle Scheme
    Based on Threshold Signature," IEEE iThings/ CPSCom 2023.
    https://ieeexplore.ieee.org/document/10501857/
55. Discreet Log Contracts specifications, "MultiOracle.md" (combining
    independent oracle attestations via t-of-t adaptor signatures).
    https://github.com/discreetlogcontracts/dlcspecs/blob/master/MultiOracle.md
56. P2PDerivatives / Crypto Garage, `p2pderivatives-oracle` reference DLC oracle
    (Go; signs BTC/USD and BTC/JPY; repo last commit 2022-05-24).
    https://github.com/p2pderivatives/p2pderivatives-oracle
57. Kormir, reference Rust DLC oracle library (`bennyhodl/dlcdevkit/kormir`;
    tagged SHA-256 per dlcspecs; monthly releases through March 2026).
    https://github.com/bennyhodl/dlcdevkit
58. Ernest Oracle (`ernest-money/ernest-oracle`); Bitcoin chain-metric DLC
    oracle delegating to Kormir; HTTP-only, no Nostr publication in-tree; on
    hiatus since 2025-06-02. https://github.com/ernest-money/ernest-oracle
59. Magnolia Financial Services, "Oracles" product page; commercial DLC oracle
    powering Lygos, attestations gated behind API key.
    https://magnolia.financial/oracles/
60. NIP-1658, asset price publishing on Nostr (proposed kinds 31892 / 1892 /
    10041); kind 1009 is informal and not in the official NIPs registry.
    https://github.com/nostr-protocol/nips/pull/1658
61. Uniswap, "v4 Truncated Oracle Hook" blog post.
    https://blog.uniswap.org/uniswap-v4-truncated-oracle-hook
62. Pyth Network Documentation, "Hermes API Instances and Providers" (public
    endpoint at hermes.pyth.network, 20 req / 10 s rate limit, third-party node
    providers Triton, P2P, extrnode, Liquify).
    https://docs.pyth.network/price-feeds/core/api-instances-and-providers/hermes
63. RedStone SDK, hardcoded DDL gateway URLs
    (`oracle-gateway-1.a. redstone.finance`,
    `oracle-gateway-2.a.redstone.finance`, `.vip` GCP mirror).
    https://github.com/redstone-finance/redstone-oracles-monorepo/blob/main/packages/sdk/src/data-services-urls.ts
64. RedStone Documentation, "Standardized Access for DeFi Interoperability"
    (Push model; permissionless relayer service).
    https://docs.redstone.finance/docs/dapps/redstone-push/
65. RedStone blog, "RedStone on Monad: The Real-Time Data Layer for High-Speed
    DeFi," 27 Nov 2025.
    https://blog.redstone.finance/2025/11/27/redstone-on-monad-the-real-time-data-layer-for-high-speed-defi/
66. Chainlink Documentation, "Data Feeds" (deviation-threshold + heartbeat
    update mechanism; per-feed values in the dashboard; integrator responsible
    for monitoring, staleness checks, and safeguards against delays and outages;
    no numeric uptime SLA stated). https://docs.chain.link/data-feeds
67. Chainlink, "Selecting Quality Data Feeds" (reliability described
    qualitatively via decentralisation at the data-source, node, and network
    levels; no availability percentage).
    https://docs.chain.link/data-feeds/selecting-data-feeds
68. RedStone, "Terms of Use" ("AS-IS AND AS-AVAILABLE BASIS"; "WE CANNOT
    GUARANTEE THE SITE WILL BE AVAILABLE AT ALL TIMES"; no liability for
    downtime). https://redstone.finance/terms-of-use
69. RedStone, "Price Feeds" product page ("0 Mispricing Events," "100+
    protocols," "80+ chains"; no numeric availability figure).
    https://www.redstone.finance/price-feeds
70. Coinmonks (Medium), "DeFi Protocols Should Migrate to Pull Oracles" (Pyth
    pull oracle maintained 99.9% uptime during Solana congestion while the push
    oracle missed updates; secondary source, directional).
    https://medium.com/coinmonks/defi-protocols-should-migrate-to-pull-oracles-for-improved-reliability-and-performance-f5ca615cddab
71. Chainlink, ETH/USD price feed (Ethereum mainnet): 0.5% deviation threshold,
    3600-second (1-hour) heartbeat.
    https://data.chain.link/feeds/ethereum/mainnet/eth-usd
