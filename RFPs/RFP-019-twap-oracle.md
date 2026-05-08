---
id: RFP-019
title: "On-Chain TWAP Oracle"
tier: L
funding: $XXXXX
status: open
dependencies: See Platform Dependencies section
category: Developer Tooling & Infrastructure
---


# RFP-019 — On-Chain TWAP Oracle

## 🧭 Overview

Build an on-chain TWAP (time-weighted average price) oracle program
for LEZ that reads pool accumulators from a LEZ DEX (RFP-004) and
exposes geometric-mean prices through a canonical oracle price
account standard, together with a circuit-breaker interface against
external price sources. The TWAP tier is the on-chain
defence-in-depth layer alongside off-chain feeds: its security
depends on DEX liquidity depth and is independent of any bridge or
off-chain publisher. This RFP covers the TWAP program, the
canonical price account standard, and the circuit-breaker interface
only. External oracle adaptors (RedStone in RFP-020, Pyth in a
future RFP) populate the same standard. The applying team should
have experience with AMM mathematics, oracle manipulation analysis,
and SVM program development.

## 🔥 Why This Matters

Every DeFi protocol on LEZ that consumes price data, including the
lending protocol ([RFP-008](./RFP-008-lending-borrowing-protocol.md)),
the reflexive stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), and any
derivatives or liquidation engines, faces the same risk: a single
oracle source under thin-liquidity conditions is cheap to
manipulate. 36 documented flash-loan oracle attacks have caused
over $418M in cumulative losses [5]. The defence is layered: combine
on-chain TWAP with an off-chain feed and flag divergence above a
threshold. Without an on-chain TWAP tier, LEZ DeFi has to trust a
single off-chain provider with no on-chain cross-check.

The TWAP tier also enables designs that depend on on-chain pair
pricing without trusting an external publisher. One example is the
LGS/LSC composite oracle path that the LSC stablecoin (RFP-013)
may choose: an external LGS/USD feed combined with an on-chain
LGS/LSC TWAP. Whether RFP-013 picks the direct LSC/USD path or the
composite path is a business decision for the implementer; either
path needs a working TWAP tier as a swappable building block.

On new chains, on-chain TWAP is vulnerable on its own: with thin
liquidity, a PoS validator controlling two consecutive blocks can
manipulate the TWAP accumulator at a cost roughly equal to the
round-trip swap fees and price impact, with no competition for the
back-run [6]. The attack cost scales linearly with pool depth, so
pools with $1M in liquidity offer far less protection than pools
with $100M. The circuit-breaker interface in this RFP exists to
bound this risk: when an external feed is registered for the same
pair, divergence above a configurable threshold flags the price as
disputed.

## 🏗 Design Rationale

### Public oracle execution

Oracle programs run as public LEZ executions with no confidential
state. Accumulator updates, TWAP computation (including the
geometric mean), and price queries are all visible to any caller.
This is intentional: oracles are a shared public good on LEZ, and
every dapp must be able to read the same canonical price.
Confidential execution is reserved for application-layer protocols
that consume oracle prices (for example, private DEX swaps in
RFP-004); the price feed itself stays public.

### Geometric mean over arithmetic mean

Uniswap v3 moved from arithmetic-mean TWAP (v2) to geometric-mean
TWAP (v3) for good reason. The geometric mean, computed via
tick-based accumulators (log-price space), is more
manipulation-resistant for multiplicative price processes: an
attacker who moves the price up by 10x in one block and back by 10x
in the next leaves no net impact on the geometric mean, whereas an
arithmetic mean would be skewed upward [9]. LEZ's TWAP oracle
should adopt the v3 approach.

### Per-block tick-delta truncation

Raw v3 accumulators record whatever tick the pool reaches at the end
of each block, which leaves the oracle exposed on thin pools at
chain launch: a single PoS multi-block validator or flash-loan
operator can move the pool by an arbitrary amount and have the full
excursion written into the accumulator. Uniswap v4's truncated
oracle hook addresses this by clamping the per-block tick delta
written to the accumulator to a fixed maximum (the v4 reference
value is 9,116 ticks per block, roughly a 2.39x price move). An
attacker therefore cannot inject more than `MAX_TICK_DELTA *
blockTime` of distortion per block regardless of how far they push
the pool, and must sustain manipulation across many blocks while
arbitrage erodes their position. See the appendix
([Uniswap v4 truncated oracle hook](../appendix/oracle-ecosystem.md#uniswap-v4-truncated-oracle-hook),
[Mitigation: tick-delta truncation](../appendix/oracle-ecosystem.md#mitigation-tick-delta-truncation-uniswap-v4))
for the full mechanism. LEZ's TWAP oracle adopts this truncation as
a hard requirement: the cost is one tick-delta comparison and clamp
per accumulator update, and it materially raises the manipulation
floor on the thin pools that LEZ will have at launch.

### Configurable cardinality

Uniswap v3 pools default to storing a single observation
(cardinality 1). Expanding the observation ring buffer to N slots
costs a one-time storage payment and enables TWAP lookback of up to
N blocks. At 12s blocks, the maximum cardinality of 65,535 provides
approximately 9 days of history [9]. Protocols can trade storage
cost for lookback depth depending on their needs: a lending
protocol may need 1 to 2 hours of history, while a governance
oracle may need 7 days.

### LEZ oracle data standard

On EVM, Chainlink's `AggregatorV3Interface` (`latestRoundData()`)
became the de facto oracle standard because Chainlink was the first
mover; Pyth, RedStone, Switchboard, and DIA all ship compatible
wrapper contracts so that consuming protocols need no code changes.
The interface has known limitations: no confidence interval, no
source identifier, confusing `answeredInRound` semantics, and
variable `decimals()` per feed.

On SVM (Solana, LEZ), no equivalent standard exists. Each oracle
provider defines its own account data layout (Pyth's `PriceAccount`,
Switchboard's `AggregatorAccountData`), so consuming programs write
per-provider integration code. This fragmentation is not an
architectural necessity; a shared account struct is feasible on SVM.

LEZ can define a canonical oracle price account structure now,
before ecosystem fragmentation sets in. The struct should include
fields that `AggregatorV3Interface` lacks: confidence interval,
source identifier, and circuit-breaker dispute status. Because
account data structures on SVM are append-friendly (a program can
add new fields at the end of the struct without breaking consumers
that read only the existing fields), the standard can evolve over
time without requiring coordinated upgrades across consuming
protocols.

The standard is defined in this RFP. External oracle adaptors
(RFP-020 RedStone, future Pyth RFP) populate the same struct so
that consuming protocols integrate once and remain agnostic to the
underlying data source. If a new oracle provider becomes available
on LEZ, it populates the same struct without requiring any change
to consuming protocols.

### Circuit-breaker interface

When both on-chain TWAP and an external feed are available for the
same pair through the canonical price account standard, the program
compares them; if divergence exceeds a configurable threshold (e.g.
5%), the price is flagged as disputed. While the dispute is active,
the unified interface returns the most recent non-disputed price
(if available within `maxAge`) or reverts. Multi-source price
validation is the production norm: Aave V3 uses Chainlink with a
configurable fallback oracle [15]; Compound V2 anchored Coinbase
reporter prices against a Uniswap V2 TWAP with a 20% divergence
tolerance [16]; MakerDAO interposes a one-hour Oracle Security
Module (OSM) delay as a manipulation circuit breaker [15a]. Most
major lending protocols use at least two tiers of price validation
[6].

#### What the production record actually shows

The production record favours **source diversification** more
clearly than it favours the specific TWAP-anchor +
divergence-flag mechanic this RFP includes. The strongest
documented saves are not divergence-flag firings; they are
incidents where a protocol consumed a robustly-aggregated
external feed that held while a single market venue dislocated:

- **Curve / Vyper exploit, 30 July 2023.** During the
  reentrancy attack, the on-chain CRV/USD price on Curve pools
  collapsed to roughly $0.08; Chainlink's CRV/USD feed,
  aggregated across CEX prices, bottomed at approximately
  $0.59. Aave and other lending markets consuming the
  Chainlink feed avoided cascade liquidation on a position
  reported in coverage at roughly $200M. This is a clean
  source-diversification save; the divergence between the
  on-chain venue and the off-chain aggregate was the signal,
  but no protocol-side divergence-flag firing is documented.
- **USDe Binance flash, 11 October 2025.** On-chain consumers
  of robust USDe pricing stayed within roughly 30 basis points
  of $1 despite a Binance wick, again driven by aggregate
  price feeds rather than a circuit breaker.
- **MakerDAO OSM delay, 31 March 2025.** The mandatory one-hour
  delay absorbed roughly $84.4M of whale positions during an
  ETH wick. Passive save: no governance `stop()` was called;
  the delay window itself was the protection.
- **Base sequencer outage, 5 August 2025.** Aave and Moonwell
  on Base used the generic Chainlink Sequencer Uptime Feed to
  pause borrowing and liquidations during a 33-minute handoff
  outage. Note that this is the generic L2 uptime feed, not
  Aave's `PriceOracleSentinel` wrapper.

The track record of the specific TWAP-anchor +
divergence-flag mechanic this RFP includes is thinner. The
canonical reference is Compound V2's `UniswapAnchoredView`
during the November 2020 DAI spike: the 20% TWAP anchor
rejected the most extreme prices but did not prevent
approximately $89M of liquidations because the tolerance was
too wide. Compound V3 then dropped the TWAP anchor entirely.
Aave removed `PriceOracleSentinel` in v3.7 after years on four
chains with no documented saves and repeated false positives.
Aave's CAPO wstETH safety wrapper itself became the failure
source on 10 March 2026, causing roughly $27M of
false-positive liquidations. No protocol publishes "divergence
flag fired and saved $N" events the way protocols publish
post-mortems for losses.

Layered defence pays off, but the load-bearing layer is
**source diversification** with robust aggregate feeds that
hold under venue-level distortion, not **bounds-checking
circuit breakers** that fire when sources disagree. This RFP's
circuit-breaker interface is worth shipping for the cases it
does catch (the bounds it imposes are a hedge against either
source going badly wrong), but the RFP-013, RFP-008, and
RFP-004 implementers should not treat the divergence flag as a
substitute for selecting feeds whose aggregation method is
itself robust. The Recommended Consumer Pattern in the SDK doc
packet should be explicit on this point: cross-check is a
guard rail, not a primary defence.

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
   cardinality: default 1, expandable up to 65,535 observations
   per pool.
3. Provide a query interface: given a pool address and a window
   length, return the TWAP price and the observation timestamps
   used.
4. Define and implement the canonical LEZ oracle price account
   structure as a reusable standard for the ecosystem (see Design
   Rationale, "LEZ oracle data standard"). The struct must include
   at minimum: price, timestamp, source identifier, confidence
   interval (where the source provides one; zero otherwise), and
   circuit-breaker dispute flag. The TWAP source must populate this
   struct. The struct must be specified as a SPEL IDL and published
   as a standalone artefact that other programs (including external
   oracle adaptors in RFP-020 and any future Pyth RFP) can import
   without depending on the TWAP program itself. The interface must
   reject any price that is zero, negative, or otherwise invalid
   before writing it to the account.
5. Implement a circuit breaker against external price sources that
   conform to the canonical price account standard: when both the
   on-chain TWAP and at least one external source are registered
   for the same pair, the program compares them; if divergence
   exceeds a configurable threshold (e.g. 5%), the program flags
   the price as disputed. While the dispute is active, the unified
   interface returns the most recent non-disputed price (if
   available within `maxAge`) or reverts if no valid non-disputed
   price exists. Consuming protocols can query the dispute status
   and act accordingly.
6. The oracle program owner can register new TWAP price feed
   sources (add a pool) and external price sources (any program
   that publishes to the canonical price account standard), and
   deregister stale or compromised sources.
7. Every price returned through the unified interface includes a
   timestamp. Consuming protocols can specify a `maxAge` parameter;
   the interface rejects prices older than `maxAge`.

#### Usability

1. Provide an SDK that can be used to build Logos modules for
   interacting with the oracle program (querying prices, expanding
   cardinality, registering feed sources).
2. Provide a Logos mini-app GUI (price feed dashboard) with local
   build instructions, downloadable assets, and loadable in Logos
   app (Basecamp) via git repo. The dashboard must display: live
   TWAP prices, TWAP versus registered external source comparison,
   circuit-breaker status, and observation history.
3. Provide a CLI that covers core functionality: query price,
   expand cardinality, register and deregister feed sources.
4. Provide an IDL for the oracle program and the oracle price
   account standard, using the
   [SPEL framework](https://github.com/logos-co/spel). The price
   account IDL must be published as a standalone artefact that
   other programs can import without depending on the oracle
   program itself.
5. Return clear, actionable error messages for all failure modes:
   stale price, disputed price (circuit breaker triggered), no
   observation history for the requested window, cardinality too
   low for the requested window, zero or negative price from
   source, and no valid non-disputed price available.
6. Provide a **reference consumer program**: a minimal LEZ program
   (or equivalently a documented program-side code snippet plus
   tests) that demonstrates the recommended consumer-side
   integration pattern for reading the canonical price account.
   The reference must show: reading price and timestamp from the
   account, rejecting prices older than the consumer's chosen
   `maxAge`, refusing to act on a price whose dispute flag is set,
   and the recommended response when a price is unavailable
   (typically: refuse the action, do not fall back to an unsafe
   default). This is a guidance artefact for downstream consumer
   protocols (RFP-008, RFP-013, RFP-004), not a production
   product on its own.

#### Reliability

1. A price query is read-only and never modifies oracle state.
2. Cardinality expansion is atomic: partial failure leaves
   existing observations intact.
3. Circuit-breaker evaluation is deterministic: given the same
   on-chain state, the same divergence result is produced.

#### Performance

1. A TWAP query completes within a single LEZ transaction.
2. Document the compute unit (CU) cost of each operation: TWAP
   query, accumulator update, cardinality expansion, and circuit
   breaker evaluation. LEZ's per-transaction compute budget may
   change during testnet.

#### Supportability

1. The oracle program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI; CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test. The test
   suite must include: TWAP computation correctness (known
   accumulator values produce expected prices), tick-delta
   truncation (a synthetic price excursion exceeding
   `MAX_TICK_DELTA` is clamped in the accumulator and the
   resulting TWAP matches the truncated trajectory, not the raw
   one), manipulation detection (circuit breaker triggers when
   TWAP and an external source diverge beyond threshold),
   staleness rejection (prices older than `maxAge` are rejected),
   and registration / dispute state transitions.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for querying prices,
   expanding cardinality, and registering feed sources via CLI and
   mini-app.
5. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for
   querying prices, expanding cardinality, and registering feed
   sources, **plus a "Recommended Consumer Pattern" section** that
   walks a downstream protocol developer through the reference
   consumer program from Usability #6: staleness handling,
   dispute-flag handling, behaviour when no valid non-disputed
   price is available, and the recommended pairing with an
   external feed for divergence checking.
6. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
7. Provide Figma designs or equivalent for the mini-app GUI (price
   feed dashboard).

#### + Oracle Security

1. The TWAP computation must sample the accumulator at block
   boundaries (before any same-block trades execute), not
   mid-block, to resist within-block manipulation.
2. The minimum recommended observation window for lending protocol
   use is documented, with a manipulation-cost analysis for
   representative LEZ liquidity levels ($1M, $10M, $50M, and $100M
   pool depth).
3. Implement per-block tick-delta truncation on accumulator updates,
   following the Uniswap v4 truncated-oracle-hook design (see
   Design Rationale, "Per-block tick-delta truncation"). Each
   accumulator update must clamp the recorded tick delta against
   the previous block's recorded tick to a configurable
   `MAX_TICK_DELTA` (default: 9,116 ticks per block, matching the
   v4 reference). The clamp must be applied before the
   `tickCumulative` update, and `MAX_TICK_DELTA` must be governable
   per pool by the oracle program owner so it can be tuned to LEZ
   block time and per-pair volatility. The manipulation-cost
   analysis required by item 2 must report costs both with and
   without truncation, so the tradeoff is explicit.

### Soft Requirements

1. Aggregator-helper SDK utility: a client-side library that
   composes the canonical price account with one or more
   external-feed accounts (registered via the same standard) and
   applies the documented Recommended Consumer Pattern from the
   SDK doc packet: staleness rejection, dispute-flag handling,
   divergence cross-check, and fail-safe behaviour when no valid
   non-disputed price is available. This reduces duplicated
   boilerplate across consumer protocols (RFP-008, RFP-013,
   RFP-004) without changing the on-chain program surface.

### Out of Scope

The following are explicitly excluded from this RFP and addressed
elsewhere:

- External oracle adaptors. RedStone is delivered in
  [RFP-020](./RFP-020-redstone-oracle-adaptor.md). A Pyth adaptor
  (which depends on Wormhole on LEZ) is deferred to a future RFP.
- Confidential or shielded oracle execution. Oracle programs run
  as public LEZ executions (see Design Rationale, "Public oracle
  execution").
- The reflexive stablecoin design. RFP-013 owns the LSC stablecoin
  and is the consumer of TWAP and external feeds; the choice
  between LSC/USD direct and LGS/USD + LGS/LSC composite is a
  business decision for that RFP's implementer.

## ⚠ Platform Dependencies

### Hard blockers

These must be available on LEZ before the corresponding features
can be developed.

#### RFP-004 (Privacy-Preserving DEX)

The TWAP oracle reads pool accumulators from the DEX. Without
RFP-004, the on-chain TWAP tier cannot be exercised. The canonical
price account standard and circuit-breaker interface can be
designed and prototyped in parallel.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission (LP-0012)

Analytics, monitoring, and the circuit-breaker dashboard benefit
from structured on-chain events for price updates, circuit-breaker
triggers, and cardinality expansions.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently
**open**.

## 👤 Recommended Team Profile

Team experienced with:

- Oracle or DeFi protocol infrastructure development
- AMM mechanics and TWAP mathematics (accumulator design,
  geometric-mean computation, window selection)
- Solana or SVM program development (Anchor or native)
- Smart-contract security auditing (oracle manipulation, flash-loan
  attack vectors)

## ⏱ Timeline Expectations

Estimated duration: **8 to 12 weeks**.

The canonical price account standard and circuit-breaker interface
can be designed and shipped early; the TWAP program itself depends
on RFP-004 (DEX) and is the longer pole.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (pool accumulators, TWAP data source)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (primary consumer of oracle price feeds)
- [RFP-012 — Advanced Lending Features](./RFP-012-advanced-lending-features.md)
  (eMode and multi-collateral require reliable oracles)
- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
  (consumer of price feeds; LGS/LSC composite oracle path depends
  on TWAP)
- [RFP-020 — RedStone Off-Chain Oracle Adaptor](./RFP-020-redstone-oracle-adaptor.md)
  (first external adaptor to the canonical price account standard)
- [Appendix: Oracle Ecosystem](../appendix/oracle-ecosystem.md)
- [Uniswap v3 Oracle Documentation](https://docs.uniswap.org/concepts/protocol/oracle)
- [Uniswap v3 TWAP Oracles in PoS](https://blog.uniswap.org/uniswap-v3-oracles)


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
