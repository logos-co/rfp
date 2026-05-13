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
account standard. The on-chain TWAP serves two roles: it is the
**only** pricing path available for LEZ-native assets (LGS, the
reflexive stablecoin)
because no off-chain publisher has data to sign for pairs that
exist only on the LEZ DEX, and it is a **defence-in-depth layer**
for wrapped external assets (wXMR, wZEC, wBTC, wETH) that are
priced primarily by external oracles. This RFP covers the TWAP
program and the canonical price account standard only. The standard
allows multiple sources to publish prices for the same pair without
merging them; cross-source validation policy is the consumer
protocol's responsibility (see Design Rationale, "Multi-source
coexistence"). External oracle adaptors (RedStone in RFP-020, Pyth
in a future RFP) populate the same standard. The applying team
should have experience with AMM mathematics, oracle manipulation
analysis, and SVM program development.

## 🔥 Why This Matters

LEZ DeFi has two distinct pricing needs, and external oracles only
solve one of them.

**LEZ-native assets have no off-chain price.** LGS (the Logos
token) and the reflexive stablecoin trade on a LEZ DEX (RFP-004)
and nowhere else at launch. No off-chain publisher (Chainlink,
Pyth, RedStone) has data to sign for these pairs because there is
no underlying CEX or institutional market to source from. The only
honest source is the LEZ DEX itself, read through an on-chain TWAP.
This is not a defence-in-depth nicety; it is the only pricing path
available for LEZ-native assets.

The reflexive stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)) makes this
concrete. Its feedback controller computes a redemption rate from
the deviation between *market price* and *redemption price*, both
denominated in the reference collateral (following the RAI design,
which uses ETH-denominated prices throughout the controller). The
market price input is therefore the reflexive stablecoin priced in
its reference collateral (a LEZ-native pair, e.g. against LGS or
wBTC) that only the LEZ DEX can price. Without an on-chain TWAP
tier, RFP-013
has no price feed to read.

**Wrapped assets need manipulation-resistant pricing.** Wrapped
external assets (wXMR, wZEC, wBTC, wETH, potentially wSOL) used as
collateral by the lending protocol
([RFP-008](./RFP-008-lending-borrowing-protocol.md)) and others are
priced primarily by external oracles (RedStone in
[RFP-020](./RFP-020-redstone-oracle-adaptor.md), Pyth in a future
RFP). These also benefit from an on-chain TWAP cross-check where a
LEZ DEX pair exists with sufficient liquidity, since 36 documented
flash-loan oracle attacks have caused over $418M in cumulative
losses [5] and a single off-chain source remains the dominant DeFi
attack vector. The canonical price account standard supports
multi-source coexistence: when both an on-chain TWAP and an
external feed exist for the same pair, each publishes its own
price account and consumer protocols apply their own cross-source
policy (which is primary, which is fallback, how to handle
divergence). This RFP intentionally does not bundle a generic
divergence policy in the oracle program; the production record
shows that policy belongs in the consumer (see Design Rationale,
"Multi-source coexistence").

**The thin-liquidity caveat applies to both paths.** On new chains,
on-chain TWAP is itself vulnerable: with thin liquidity, a PoS
validator controlling two consecutive blocks can manipulate the
TWAP accumulator at a cost roughly equal to the round-trip swap
fees and price impact, with no competition for the back-run [6].
The attack cost scales linearly with pool depth, so pools with $1M
in liquidity offer far less protection than pools with $100M. This
RFP mitigates the thin-pool case in two ways: per-block tick-delta
truncation (Uniswap v4 style; see Design Rationale below) raises the
single-block manipulation floor, and minimum-window guidance with
explicit manipulation-cost analysis tells consumer protocols how
long a window they need at LEZ launch liquidity levels.

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
fields that `AggregatorV3Interface` lacks: explicit base and quote
asset identifiers, confidence interval, and source identifier.
Because
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

### Multi-source coexistence

When the same pair has both an on-chain TWAP and one or more
external feeds, each source publishes its own price account under
the canonical struct. The oracle program does not merge or compare
them. Cross-source policy (which feed is primary, which is
fallback, what to do when sources disagree) is the consumer
protocol's responsibility, because risk tolerance and acceptable failure
modes vary by consumer: a lending protocol pausing liquidations is not
the same decision as a stablecoin pausing rate updates.

The production record supports this split. The strongest
documented saves come from **source diversification** combined
with **consumer-owned policy**, not from generic divergence-flag
modules:

- **Curve / Vyper exploit, 30 July 2023.** During the
  reentrancy attack, on-chain CRV/USD on Curve pools collapsed
  to roughly $0.08; Chainlink's aggregated CRV/USD bottomed at
  approximately $0.59. Aave and other lending markets consuming
  the Chainlink feed avoided cascade liquidation on a position
  reported in coverage at roughly $200M.
- **USDe Binance flash, 11 October 2025.** On-chain consumers
  of robust USDe pricing stayed within roughly 30 basis points
  of $1 despite a Binance wick.
- **MakerDAO OSM delay, 31 March 2025.** The mandatory one-hour
  delay absorbed roughly $84.4M of whale positions during an
  ETH wick. Passive save: the delay window itself was the
  protection.
- **Base sequencer outage, 5 August 2025.** Aave and Moonwell
  on Base used the generic Chainlink Sequencer Uptime Feed to
  pause borrowing and liquidations during a 33-minute handoff.

By contrast, generic TWAP-anchor + divergence-flag modules have a
notably negative record: Compound V2's `UniswapAnchoredView` did
not prevent approximately $89M of liquidations during the
November 2020 DAI spike (Compound V3 then dropped it); Aave
removed `PriceOracleSentinel` in v3.7 after years with no
documented saves and repeated false positives; Aave's CAPO wstETH
safety wrapper itself caused roughly $27M of false-positive
liquidations on 10 March 2026.

The load-bearing layers in production are source diversification,
time delay (OSM-style), and consumer-specific policy, all of which
are owned by the consumer. RFP-019's job is to deliver the
on-chain TWAP feed and the canonical struct that lets multiple
feeds coexist. Bundling a generic divergence policy in the oracle
program would also introduce a **liveness griefing surface**: an
attacker manipulating a thin LEZ DEX pool (cheap at chain launch,
per Oracle Security #2) could trigger a global halt for every
consumer through a shared gate. Implementing protocols (RFP-008,
RFP-013, RFP-004) own their multi-source policy in their own
scope of work. The Recommended Consumer Pattern in the SDK doc
packet (Supportability #5) provides a canonical example that
consumers can adopt or replace.

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
   at minimum: `base_asset`, `quote_asset`, price, timestamp, source
   identifier, and confidence interval (where the source provides
   one; zero otherwise). `base_asset` and `quote_asset` are
   canonical asset identifiers (token mint addresses for
   LEZ-resident tokens; agreed canonical IDs for off-chain assets
   such as `USD`) and together specify the denomination of `price`
   (i.e. the value is *base per quote*, or equivalently, how much
   of `quote_asset` one unit of `base_asset` is worth). Every price
   account must populate these fields, and consuming protocols must
   verify them before reading the price. The TWAP source must
   populate this struct. The struct must be specified as a SPEL IDL
   and published as a standalone artefact that other programs
   (including external oracle adaptors in RFP-020 and any future
   Pyth RFP) can import without depending on the TWAP program
   itself. The interface must reject any price that is zero,
   negative, or otherwise invalid before writing it to the account.
   Multiple sources may publish price accounts for the same
   `(base_asset, quote_asset)` pair; the standard does not merge
   them, and cross-source policy is owned by consuming protocols
   (see Design Rationale, "Multi-source coexistence").
5. The oracle program owner can register new TWAP price feed
   sources (add a pool) for which this RFP's program is responsible
   for accumulator reads and price-account writes. External price
   sources (RedStone in RFP-020, future Pyth RFP) publish their own
   price accounts directly under the canonical standard and are not
   registered with this program.
6. Every price account includes a timestamp. The query interface
   accepts a `maxAge` parameter; reads older than `maxAge` return
   an error rather than a stale value, leaving the consumer free
   to decide what to do (revert, fall back to another source, log
   and continue).

#### Usability

1. Provide an SDK that can be used to build Logos modules for
   interacting with the oracle program (querying prices, expanding
   cardinality, registering feed sources).
2. Provide a Logos mini-app GUI (price feed dashboard) with local
   build instructions, downloadable assets, and loadable in Logos
   app (Basecamp) via git repo. The dashboard must display: live
   TWAP prices for each registered pool, side-by-side comparison
   against any other price account published for the same
   `(base_asset, quote_asset)` pair (e.g. RedStone), and observation
   history.
3. Provide a CLI that covers core functionality: query price,
   expand cardinality, register and deregister feed sources.
4. Provide an IDL for the oracle program and the oracle price
   account standard, using the
   [SPEL framework](https://github.com/logos-co/spel). The price
   account IDL must be published as a standalone artefact that
   other programs can import without depending on the oracle
   program itself.
5. Return clear, actionable error messages for all failure modes:
   stale price, no observation history for the requested window,
   cardinality too low for the requested window, zero or negative
   price from source, and `(base_asset, quote_asset)` mismatch
   between the consumer's expected pair and the price account
   read.
6. Provide a **reference consumer program**: a minimal LEZ program
   (or equivalently a documented program-side code snippet plus
   tests) that demonstrates the recommended consumer-side
   integration pattern for reading the canonical price account.
   The reference must show: verifying the `(base_asset,
   quote_asset)` pair matches the consumer's expectation; reading
   price, timestamp, and confidence interval from the account;
   rejecting prices older than the consumer's chosen `maxAge`; and
   the recommended response when a price is unavailable (typically:
   refuse the action, do not fall back to an unsafe default). The
   reference must also include a worked **multi-source pattern**: a
   consumer that reads two price accounts for the same pair (e.g.
   on-chain TWAP and RedStone) and applies an example consumer
   policy (primary feed with fallback on staleness, plus a
   divergence cross-check that logs but does not gate). The
   reference makes clear that this policy is illustrative; each
   downstream protocol owns its own choice. This is a guidance
   artefact for downstream consumer protocols (RFP-008, RFP-013,
   RFP-004), not a production product on its own.

#### Reliability

1. A price query is read-only and never modifies oracle state.
2. Cardinality expansion is atomic: partial failure leaves
   existing observations intact.
3. Failure of one registered pool (e.g. accumulator unavailable,
   cardinality exhausted) does not affect price queries against
   other registered pools.

#### Performance

1. A TWAP query completes within a single LEZ transaction.
2. Document the compute unit (CU) cost of each operation: TWAP
   query, accumulator update, and cardinality expansion. LEZ's
   per-transaction compute budget may change during testnet.

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
   one), staleness rejection (prices older than `maxAge` are
   rejected), `(base_asset, quote_asset)` mismatch rejection, and
   pool registration / deregistration state transitions.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for querying prices,
   expanding cardinality, and registering feed sources via CLI and
   mini-app.
5. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for
   querying prices, expanding cardinality, and registering feed
   sources, **plus a "Recommended Consumer Pattern" section** that
   walks a downstream protocol developer through the reference
   consumer program from Usability #6: `(base_asset, quote_asset)`
   verification, staleness handling, behaviour when a price is
   unavailable, and an example multi-source policy (primary feed
   plus fallback, with a divergence cross-check that logs but does
   not gate). The pattern is illustrative; the doc must state
   explicitly that cross-source policy is owned by the consumer
   protocol, not by this oracle program.
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

1. Multi-source consumer-pattern helper SDK: a library implementing
   the example multi-source policy from the SDK doc packet
   (primary feed plus fallback on staleness, with a divergence
   cross-check that logs but does not gate). The helper reads two
   or more price accounts for the same `(base_asset, quote_asset)`
   pair and returns a single result plus diagnostic flags
   (divergence detected, fallback in use). Consumer protocols are
   free to use this helper as-is, configure it differently, or
   replace it with their own policy: cross-source policy is owned
   by the consumer, and this helper is one canonical implementation
   among possible others. Reduces duplicated boilerplate across
   consumer protocols (RFP-008, RFP-013, RFP-004) without changing
   the on-chain program surface.

### Out of Scope

The following are explicitly excluded from this RFP and addressed
elsewhere:

- External oracle adaptors. RedStone is delivered in
  [RFP-020](./RFP-020-redstone-oracle-adaptor.md). A Pyth adaptor
  (which depends on Wormhole on LEZ) is deferred to a future RFP.
- Confidential or shielded oracle execution. Oracle programs run
  as public LEZ executions (see Design Rationale, "Public oracle
  execution").
- The reflexive stablecoin design. RFP-013 owns the reflexive
  stablecoin and is a consumer of this oracle. RFP-013's controller
  reads the stablecoin priced in its reference collateral (a
  LEZ-native pair, e.g. against LGS or wBTC), which this RFP
  delivers via on-chain TWAP. RFP-013's choice of reference
  collateral, controller parameters, and any auxiliary
  USD-denominated reporting are owned by that RFP, not this one.
- Price feed composition (combining two or more price accounts
  whose denominations chain together, e.g.
  `LGS/USD = LGS/wBTC × wBTC/USD`). This RFP exposes the
  `base_asset` and `quote_asset` fields that make composition
  *checkable*, but does not specify or implement the composition
  itself. Composition becomes relevant only once token wrapping is
  defined on LEZ. A future RFP, likely an evolution of
  [RFP-020](./RFP-020-redstone-oracle-adaptor.md) (or a dedicated
  token-wrapping RFP), is expected to specify a canonical
  composition pattern including confidence-interval and staleness
  rules across legs. Until then, consumer protocols that need a
  price in a denomination not directly published are responsible
  for their own composition logic.

## ⚠ Platform Dependencies

### Hard blockers

These must be available on LEZ before the corresponding features
can be developed.

#### RFP-004 (Privacy-Preserving DEX)

The TWAP oracle reads pool accumulators from the DEX. Without
RFP-004, the on-chain TWAP tier cannot be exercised. The canonical
price account standard can be designed and prototyped in parallel.

### Soft blockers

Desirable but the RFP can open without them.

None

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

The canonical price account standard can be designed and shipped
early; the TWAP program itself depends on RFP-004 (DEX) and is the
longer pole.

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
  (consumer of price feeds; the RAI-style controller reads the
  reflexive stablecoin priced in its reference collateral, a
  LEZ-native pair only the on-chain TWAP can price)
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
