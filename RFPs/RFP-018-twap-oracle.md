---
id: RFP-018
title: "TWAP Oracle and Price Feed Infrastructure"
tier: L
funding: $XXXXX
status: open
dependencies: See Platform Dependencies section
category: Developer Tooling & Infrastructure
---


# RFP-018 — TWAP Oracle and Price Feed Infrastructure

## 🧭 Overview

Build a two-tier oracle system for LEZ: an on-chain TWAP (time-weighted
average price) oracle program that computes prices from LEZ DEX pool
accumulators, plus integration adaptors for external oracle feeds (Pyth,
RedStone). Every DeFi protocol on LEZ (lending, derivatives,
liquidations, stablecoins) requires reliable price feeds to function.
The oracle ecosystem secures approximately $138B in total value secured
(TVS) across chains: Chainlink alone secures $93B, Pyth $8.6B across
81 chains, and RedStone $7.2B across 110+ chains [1][2][3]. On a new
chain with thin liquidity, on-chain TWAP alone is insufficient because
manipulation cost scales linearly with pool depth; external oracle feeds
from established networks provide the safety baseline from day one. The
applying team should have experience with oracle or DeFi infrastructure
development, AMM mathematics, and cryptographic verification.

## 🔥 Why This Matters

Without price feeds, the lending protocol (RFP-008), liquidation
engines, and any derivatives built on LEZ cannot function. Price
oracles are the single most critical shared dependency across DeFi
applications.

On new chains, on-chain TWAP oracles are acutely vulnerable: with thin
liquidity, a PoS validator controlling two consecutive blocks can
manipulate the TWAP accumulator at a cost of only 2x pool fees, with
no competition for the back-run [4]. The attack cost scales linearly
with pool depth, so pools with $1M in liquidity offer far less
protection than pools with $100M. Historically, 36 documented flash
loan oracle attacks have caused over $418M in cumulative losses [5].

LEZ needs external oracle feeds (Pyth, RedStone) from day one; on-chain
TWAP grows in reliability as DEX liquidity deepens. The circuit breaker
(comparing on-chain TWAP to an external feed and flagging divergence)
provides defence in depth: neither tier alone is sufficient, but
together they create a layered security model where each compensates
for the other's weaknesses.

## Design Rationale

### Two-tier architecture

On-chain TWAP and external oracle feeds serve complementary roles.
TWAP is trustless and requires no off-chain infrastructure, but its
security depends entirely on pool liquidity depth. External oracles
(Pyth, RedStone) aggregate prices from dozens of first-party data
publishers across centralised and decentralised exchanges, providing
manipulation resistance independent of on-chain liquidity. Neither
tier is sufficient alone: TWAP fails on thin-liquidity pools, and
external oracles introduce off-chain trust assumptions. Combining
both tiers (with a circuit breaker that compares them) is the
production best practice used by Aave, MakerDAO, and other major
lending protocols [6][7].

### Pull model over push

Push oracles (Chainlink's traditional model) require dedicated node
operators to submit updates on a heartbeat or deviation threshold,
consuming gas regardless of whether any protocol reads the price. On
a new chain with low initial TVL, this creates the chicken-and-egg
problem: node operators need economic incentives that only emerge with
TVL, but DeFi needs oracles to attract TVL [8]. Pull oracles (Pyth,
RedStone) shift the cost to consumers, who fetch and verify signed
price data at transaction time. This model works from day one with
zero dedicated oracle infrastructure on the chain.

### Geometric mean over arithmetic mean

Uniswap v3 moved from arithmetic mean TWAP (v2) to geometric mean
TWAP (v3) for good reason. The geometric mean, computed via tick-based
accumulators (log-price space), is more manipulation-resistant for
multiplicative price processes: an attacker who moves the price up by
10x in one block and back by 10x in the next leaves no net impact on
the geometric mean, whereas an arithmetic mean would be skewed upward
[9]. LEZ's TWAP oracle should adopt the v3 approach.

### Configurable cardinality

Uniswap v3 pools default to storing a single observation (cardinality
1). Expanding the observation ring buffer to N slots costs a one-time
storage payment and enables TWAP lookback of up to N blocks. At 12s
blocks, the maximum cardinality of 65,535 provides approximately 9
days of history [9]. Protocols can trade storage cost for lookback
depth depending on their needs: a lending protocol may need 1 to 2
hours of history, while a governance oracle may need 7 days.

### AggregatorV3Interface compatibility

Chainlink's AggregatorV3Interface (`latestRoundData()`) is the
industry standard consumed by hundreds of DeFi protocols. Wrapping
all oracle sources (TWAP, Pyth, RedStone) behind a unified interface
means consuming protocols can integrate once and remain agnostic to
the underlying data source. If a new oracle provider becomes available
on LEZ, it can be added behind the same interface without requiring
any change to consuming protocols.

### Fee structure

Proposals must specify a fee model covering: who pays for oracle
updates (consumer, protocol, or subsidised), when fees are charged
(per query, per update, or per registration), the fee rate or
formula, and where fees are routed (protocol treasury, oracle
operators, or burned). The fee model should be sustainable without
ongoing subsidies once LEZ reaches moderate TVL.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement an on-chain TWAP oracle program that reads pool
   accumulators from a LEZ DEX (RFP-004) and computes the geometric
   mean TWAP over a configurable observation window.
2. Implement tick-based accumulator storage with configurable
   cardinality: default 1, expandable up to 65,535 observations per
   pool.
3. Provide a query interface: given a pool address and a window
   length, return the TWAP price and the observation timestamps
   used.
4. Implement an external oracle adaptor for Pyth: verify Wormhole
   VAA and Merkle proof, extract price and confidence interval, and
   reject the update if the confidence interval exceeds a
   configurable threshold or the price is stale (configurable
   maxAge).
5. Implement an external oracle adaptor for RedStone: verify node
   signatures from calldata, extract price, and reject the update
   if the price is stale (configurable maxAge).
6. Provide a unified price feed interface (AggregatorV3Interface or
   equivalent): consuming protocols query a single interface that
   returns price, timestamp, and source identifier, regardless of
   the underlying oracle source.
7. Implement a circuit breaker: when both on-chain TWAP and an
   external feed are available for the same pair, the program
   compares them; if divergence exceeds a configurable threshold
   (e.g. 5%), the program flags the price as disputed. Consuming
   protocols can query the dispute status and act accordingly.
8. The oracle program owner can register new price feed sources (add
   a pool for TWAP, register an external oracle adaptor) and
   deregister stale or compromised sources.
9. Every price returned through the unified interface includes a
   timestamp. Consuming protocols can specify a maxAge parameter;
   the interface rejects prices older than maxAge.

#### Usability

1. Provide an SDK that can be used to build Logos modules for
   interacting with the oracle program (querying prices, expanding
   cardinality, registering feed sources).
2. Provide a Logos mini-app GUI (price feed dashboard) with local
   build instructions, downloadable assets, and loadable in Logos
   app (Basecamp) via git repo. The dashboard must display: live
   prices, TWAP vs external oracle comparison, circuit breaker
   status, and observation history.
3. Provide a CLI that covers core functionality: query price, expand
   cardinality, register and deregister feed sources.
4. Provide an IDL for the oracle program, preferably using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Return clear, actionable error messages for all failure modes:
   stale price, disputed price (circuit breaker triggered), no
   observation history for the requested window, cardinality too
   low for the requested window, and invalid cryptographic proof.

#### Reliability

1. A price query is read-only and never modifies oracle state.
2. Cardinality expansion is atomic: partial failure leaves existing
   observations intact.
3. Circuit breaker evaluation is deterministic: given the same
   on-chain state, the same divergence result is produced.

#### Performance

1. A TWAP query completes within a single LEZ transaction.
2. Document the compute unit (CU) cost of each operation: TWAP
   query, Pyth VAA verification, RedStone signature verification,
   and cardinality expansion. LEZ's per-transaction compute budget
   may change during testnet.
3. External oracle verification (Pyth VAA + Merkle proof, RedStone
   signature check) must complete within a single LEZ transaction.

#### Supportability

1. The oracle program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI; CI must be green on
   the default branch.
3. Every hard requirement has at least one corresponding test. The
   test suite must include: TWAP computation correctness (known
   accumulator values produce expected prices), manipulation
   detection (circuit breaker triggers when TWAP and external feed
   diverge beyond threshold), staleness rejection (prices older
   than maxAge are rejected), Pyth VAA verification (valid and
   invalid signatures), and RedStone signature verification (valid
   and invalid signatures).
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for querying prices,
   expanding cardinality, and registering feed sources via CLI and
   mini-app.

#### + Oracle Security

1. The TWAP computation must sample the accumulator at block
   boundaries (before any same-block trades execute), not mid-block,
   to resist within-block manipulation.
2. The minimum recommended observation window for lending protocol
   use is documented, with a manipulation cost analysis for
   representative LEZ liquidity levels ($1M, $10M, $50M, and $100M
   pool depth).
3. The adaptor for external oracles must validate all cryptographic
   proofs (Wormhole guardian signatures for Pyth, node signatures
   for RedStone) and reject unsigned or incorrectly signed data.

### Soft Requirements

1. Switchboard TEE-based oracle adaptor (Trusted Execution
   Environment security model; permissionless feed creation).
2. DIA Lumina ZK-verified adaptor (fully permissionless data
   sourcing and feed deployment).
3. Multi-source aggregation: compute the median of N sources (e.g.
   TWAP + Pyth + RedStone) as a single aggregated price, reducing
   reliance on any one source.
4. Ormer algorithm implementation: median estimator with
   multi-window fusion as an alternative to standard geometric mean
   TWAP, if a production-ready specification exists. The Ormer
   paper (2024) reports 15.3% lower mean absolute error, 49.3%
   lower delay, and 15.2% lower gas cost compared to TWAP [5].
5. Historical price API: query past prices by timestamp or block
   range for analytics and backtesting.

## ⚠ Platform Dependencies

### Hard blockers

These must be available on LEZ before the corresponding features can
be developed.

#### RFP-004 (Privacy-Preserving DEX)

The TWAP oracle reads pool accumulators from the DEX. Without RFP-004,
only external oracle adaptors (Pyth, RedStone) can function. The
external adaptor tier is independently useful and can ship before
RFP-004 is delivered.

#### General cross-program calls (LP-0015)

The oracle program must call the DEX program to read pool accumulators.
[LP-0015](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0015.md)
(General cross-program calls via tail calls) is currently **open**.

#### On-chain clock / timestamp

TWAP computation requires block timestamps to calculate the time delta
between observations. Interest accrual in the lending protocol
(RFP-008) has the same dependency.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission (LP-0012)

Analytics, monitoring, and the circuit breaker dashboard benefit from
structured on-chain events for price updates, circuit breaker triggers,
and cardinality expansions.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

#### Wormhole integration on LEZ

The Pyth adaptor requires Wormhole VAA verification. Without Wormhole
on LEZ, only the RedStone adaptor (which requires no bridge) can
provide external oracle data. RedStone alone provides sufficient
coverage for launch; Pyth can be added once Wormhole is available.

## 👤 Recommended Team Profile

Team experienced with:

- Oracle or DeFi protocol infrastructure development
- AMM mechanics and TWAP mathematics (accumulator design, geometric
  mean computation, window selection)
- Cryptographic verification (Wormhole VAA signature schemes, Merkle
  proofs, RedStone node signatures)
- Solana or SVM program development (Anchor or native)
- Smart contract security auditing (oracle manipulation, flash loan
  attack vectors)

## ⏱ Timeline Expectations

Estimated duration: **10 to 14 weeks**.

The external oracle adaptors (Pyth, RedStone) and the unified
interface can ship independently of the on-chain TWAP tier, which
depends on RFP-004 (DEX). A phased delivery is expected: external
adaptors first, TWAP second.

## Evaluation Criteria

| Criterion | Weight | What we look for |
|-----------|--------|------------------|
| Technical design quality | 30% | TWAP accumulator correctness, manipulation cost analysis, AggregatorV3Interface design |
| Oracle security | 25% | Circuit breaker design, cryptographic verification completeness, staleness handling |
| Team experience | 20% | Prior oracle or DeFi infrastructure work, security track record |
| Timeline and milestones | 15% | Realistic schedule, phased delivery (external adaptors first, TWAP second) |
| Ecosystem alignment | 10% | Open source, composability with DEX (RFP-004) and lending (RFP-008) |

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (pool accumulators, TWAP data source)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (primary consumer of oracle price feeds)
- [RFP-012 — Advanced Lending Features](./RFP-012-advanced-lending-features.md)
  (eMode and multi-collateral require reliable oracles)
- [Appendix: Oracle Ecosystem](../appendix/oracle-ecosystem.md)
- [Uniswap v3 Oracle Documentation](https://docs.uniswap.org/concepts/protocol/oracle)
- [Pyth Network EVM Integration](https://docs.pyth.network/price-feeds/core/use-real-time-data/pull-integration/evm)
- [RedStone Oracle Documentation](https://docs.redstone.finance/)
- [Uniswap v3 TWAP Oracles in PoS](https://blog.uniswap.org/uniswap-v3-oracles)


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
