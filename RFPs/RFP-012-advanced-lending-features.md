---
id: RFP-012
title: Advanced Lending Features
tier: L
funding: $XXXXX
status: open
dependencies: RFP-008
category: Applications & Integrations
---


# RFP-012 — Advanced Lending Features

## 🧭 Overview

Extend the lending protocol delivered by [RFP-008](./RFP-008-lending-borrowing-protocol.md)
with Aave v3-equivalent features: efficiency mode (eMode), isolated
markets, flash loans, multiple collateral per position, and curated
vault abstractions. These features either depend on platform primitives
not yet available on LEZ or add complexity best tackled after a
baseline protocol is live and battle-tested.

This RFP is explicitly a follow-up to RFP-008. Teams should not apply
until the RFP-008 protocol is deployed and the platform dependencies
listed below are resolved. The applying team should ideally be the same
team that delivered RFP-008, or have deep familiarity with the deployed
protocol's internals.

## 🔥 Why This Matters

Aave v3's efficiency mode enabled LST/underlying pairs to operate at
up to 97% LTV, dramatically improving capital efficiency for correlated
assets. Isolated markets allowed the ecosystem to list long-tail assets
without exposing the core protocol to unbounded risk. Flash loans
unlocked a class of atomic arbitrage and self-liquidation tooling that
deepened market efficiency. Together, these features transformed a
functional lending protocol into a composable capital market primitive.

For the Logos ecosystem, delivering these features on top of the
RFP-008 baseline will complete the capital-efficiency stack, attract
more sophisticated DeFi builders, and position LEZ as a credible
alternative to established chains for advanced lending use cases.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. **Flash loans** — uncollateralised loans borrowed and repaid within
   a single transaction. The lending program releases funds, tail-calls
   the borrower's callback (via LP-0015's capability-protected
   continuation model), and the callback must invoke `repay` before
   returning control. If the callback fails or does not repay, the
   transaction reverts atomically and no funds leave the pool. Borrowers
   pay a configurable flash loan fee.

2. **Efficiency mode (eMode)** — define correlated asset groups
   (e.g. stablecoin pairs, LST/underlying) with elevated LTV and
   liquidation threshold ratios. Borrowers automatically receive eMode
   parameters when all collateral and debt assets belong to the same
   eMode category. eMode categories are configurable by an admin
   authority.

3. **Isolated markets** — an admin authority can list higher-risk
   assets in isolated pools with independent debt ceilings. Bad debt
   from an isolated market does not propagate to cross-market pools.
   Each isolated asset has its own debt ceiling set and enforced
   on-chain by an admin authority.

4. **Multiple collateral per position** — a single borrower can
   deposit multiple distinct assets as collateral. Aggregate borrowing
   power is the weighted sum of each collateral asset's contribution
   (LTV × USD value). When interacting via a private account, the ZKP
   must verify the aggregate collateral constraint.

5. **All risk parameters adjustable without program upgrade** — every
   configurable parameter (interest rate model slope, eMode LTV,
   isolated market debt ceiling, flash loan fee, reserve factor, etc.)
   must be updatable by an admin authority without redeploying the
   program.

#### Usability

1. All new features are exposed in the SDK, CLI, and mini-app built
   in RFP-008; no separate front-end is required, only extension of
   the existing one.
2. The mini-app clearly surfaces eMode status per position (active
   category, effective LTV) and isolated market constraints (debt
   ceiling remaining).
3. Flash loan callbacks must be documented with worked examples
   showing how to implement an atomic arbitrage or self-liquidation
   flow.

#### Reliability

1. Flash loans revert atomically if the borrower's callback fails to
   invoke `repay` or the repaid amount (principal + fee) does not
   match the borrowed amount.
2. eMode LTV parameters cannot be set above a hard-coded maximum
   (configurable at deploy time, not upgradeable) to prevent
   governance from enabling 100% LTV positions.
3. Isolated market debt ceilings are enforced atomically; concurrent
   borrows cannot jointly exceed the ceiling.

#### Performance

1. Document the transaction size of each new operation (flash loan
   initiation, eMode position open, isolated market borrow) on LEZ.
   LEZ's block size is limited and this budget may change during
   testnet.
2. Multiple collateral positions must not exceed LEZ compute limits
   for a position with up to 5 collateral assets.

#### Supportability

1. The updated program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI — CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
4. A README addendum documents the new features: configuration steps,
   eMode category setup, isolated market listing process, and a
   flash loan integration guide.
5. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for each new SDK feature (flash loans, eMode, isolated markets, multiple
   collateral), covering the developer integration journey.
6. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI additions covering the new features.
7. Provide Figma designs or equivalent for all GUI additions to the mini-app.

### Soft Requirements

If possible.

#### Functionality

1. **Curated vault abstraction** — single-asset deposit vaults that
   allocate across multiple lending markets via a curator-defined
   strategy. Depositors receive vault shares; the curator rebalances
   allocations to maximise yield within risk limits they define.

#### Reliability

1. **Multi-oracle redundancy for eMode assets** — eMode correlated
   asset groups should support a secondary oracle per asset, with
   automatic fallback when the primary is stale, to prevent eMode
   positions from becoming un-liquidatable due to oracle downtime.

#### Supportability

1. **Formal verification of eMode invariants** — (a) an eMode
   position with health factor ≥ 1 cannot be liquidated, (b) eMode
   LTV is never higher than the hard-coded maximum.

### Out of Scope

- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Any feature already delivered by RFP-008

## ⚠ Platform Dependencies

This RFP is open for proposal submission. However, development is blocked until the following is satisfied:

1. **RFP-008 is live on LEZ mainnet/testnet** — this RFP extends the
   deployed protocol; it cannot proceed without the base layer.

All other platform primitives required by this RFP (including
LP-0015 general cross-program calls, oracle provider, and on-chain
clock) are hard blockers for RFP-008 and will therefore be resolved
before this RFP opens.

### Private Account Complexity

eMode (F2) and multiple collateral per position (F4) increase ZKP
complexity for private account users:

- **eMode** — the ZKP must verify that all collateral and debt assets
  belong to the same eMode category and apply the correct elevated LTV.
  This adds circuit constraints but is tractable in principle.
- **Multiple collateral** — the ZKP must aggregate the USD value of
  multiple collateral assets, each with its own oracle price and LTV.
  Circuit size grows with the number of collateral assets. Benchmarking
  proof generation time for 5-asset positions is required before
  marking this feature production-ready.

## 👤 Recommended Team Profile

Team experienced with:

- Aave v3 or equivalent protocol internals (eMode, isolated markets,
  flash loan architecture)
- Solana or SVM program development (Anchor or native)
- ZKP circuit design (for private account compatibility of multi-asset
  positions)
- Capability-based cross-program call patterns (flash loan callback model)
- DeFi governance and parameter management

Ideally the same team that delivered RFP-008.

## ⏱ Timeline Expectations

Estimated duration: **12 weeks** (after platform dependencies are
resolved and RFP-008 is live).


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- TODO: Oracle integration guide for LEZ
- TODO: SPEL framework documentation


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
