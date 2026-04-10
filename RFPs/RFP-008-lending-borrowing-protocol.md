---
id: RFP-008
title: Lending & Borrowing Protocol
tier: XL
funding: $XXXXX
status: open
dependencies: See Appendix — Platform Dependencies
category: Applications & Integrations
---


# RFP-008 — Lending & Borrowing Protocol

## 🧭 Overview

Build a permissionless, isolated-market lending protocol on LEZ,
following the Morpho Blue design: each market is an independent lending
pair defined by five immutable parameters (loan token, collateral
token, oracle, interest rate model, liquidation LTV). Anyone can create
a market from enabled parameters; the core protocol is minimal and
governance-free. The protocol includes algorithmic interest rates,
permissionless liquidation, oracle-fed price data, and flash loans via
callbacks.

Lending is the capital-efficiency engine of any DeFi ecosystem. It
unlocks idle assets by letting holders earn yield and borrowers access
liquidity without selling. Morpho Blue demonstrated the power of this
minimal, isolated-market approach: its core contract is approximately
650 lines of Solidity, yet the protocol attracted [DATA NEEDED] in
deposits within its first year on Ethereum. For the Logos ecosystem, a
lending protocol creates demand for assets deployed on LEZ, drives TVL,
and provides a composability surface that other programs can build on.

This RFP targets Morpho Blue core equivalence: the foundational
isolated-market lending primitives (market creation, supply, borrow,
repay, withdraw, liquidation, interest accrual, oracle integration,
flash loans). The curation layer (single-asset deposit vaults that
allocate across multiple markets, vault share tokens, supply and borrow
caps) is addressed in
[RFP-012](./RFP-012-advanced-lending-features.md). The applying team
should have experience building or contributing to on-chain lending
protocols and be comfortable with interest rate modelling, liquidation
mechanics, and oracle integration.

## 🔥 Why This Matters

Lending protocols are proven ecosystem catalysts. Aave's deployment on
Polygon catalysed over $1B in TVL and bootstrapped the entire Polygon
DeFi ecosystem. On Solana, Kamino grew to $2.8B TVL and became the
backbone for leveraged yield strategies across the chain.

Without lending, assets on LEZ sit idle. Holders have no way to earn
yield, builders have no liquidity primitive to compose with, and the
ecosystem lacks the capital-efficiency layer needed to attract serious
DeFi activity. A lending protocol is the single highest-leverage
application for growing the Logos ecosystem.

The Morpho Blue model offers specific advantages for a nascent
ecosystem like LEZ. Permissionless market creation removes governance
bottlenecks for listing new assets, letting the market decide which
pairs are worth lending against. Isolated markets eliminate systemic
contagion: bad debt in one market cannot cascade to others. And the
minimal core (no upgradeable parameters, no complex shared-pool
accounting) is easier to audit, formally verify, and reason about,
which is critical when the protocol is the first lending primitive on a
new chain.

Beyond TVL, lending creates downstream demand: it establishes a
borrowing market for stablecoins, generates liquidation volume for
DEXs, and provides the collateral infrastructure needed for synthetic
assets and structured products. Every major DeFi ecosystem was built on
this foundation.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Anyone can create a lending market by specifying five parameters:
   loan token mint, collateral token mint, oracle program, interest
   rate model (IRM) program, and liquidation loan-to-value ratio
   (LLTV). The market ID is the deterministic hash of these five
   parameters. Market creation is permissionless; no admin approval is
   required.
2. Users can supply the loan token to a specific market. The protocol
   tracks supply shares internally per market per account (no receipt
   token at the core level). Supply shares represent a proportional
   claim on the market's loan token balance plus accrued interest.
3. Users can withdraw supplied loan tokens from a market at any time,
   subject to available liquidity in that market.
4. Users can deposit the market's collateral token and borrow the loan
   token, up to the market's LLTV ratio.
5. Users can repay borrowed loan tokens in full or partially.
6. Each market uses the IRM program specified at creation. The IRM
   computes the borrow rate as a deterministic function of market
   utilisation. Interest accrues continuously; supply APY and borrow
   APR are queryable on-chain per market.
7. Interest accrual is lazy (computed on interaction), not via a
   separate crank transaction per block.
8. When a borrower's position LTV exceeds the market's LLTV, any
   account can permissionlessly liquidate the position by repaying a
   portion of the debt and receiving equivalent collateral plus a
   liquidation incentive. The liquidation incentive factor is derived
   from the market's LLTV (higher LLTV markets have smaller incentives;
   lower LLTV markets have larger incentives).
9. An admin authority manages two protocol-level functions:
   (a) enabling LLTV values that market creators can select from
   (e.g. 86%, 91.5%, 94.5%), and (b) enabling IRM implementations
   that market creators can select from. Market creators choose from
   enabled parameters; the admin does not control individual markets
   after creation.
10. An optional protocol fee diverts a configurable fraction of
    interest accrued in each market to a fee recipient account. The
    fee percentage and recipient are set by the admin authority.
11. Each market references the oracle program specified at creation for
    collateral valuation and liquidation triggers. The oracle is a
    separate LEZ program.
12. Flash loans: users can borrow assets from any market within a
    single transaction via a callback mechanism. The lending program
    releases funds, tail-calls the borrower's callback (via LP-0015's
    capability-protected continuation model), and the callback must
    invoke repayment before the transaction completes. If the callback
    fails or does not repay the full amount plus fee, the transaction
    reverts atomically and no funds leave the market. The flash loan
    fee is configurable by the admin authority.
13. A liquidator bot that continuously monitors all markets and
    executes liquidations when positions exceed their market's LLTV.
    The bot is the protocol's solvency mechanism; without it, bad debt
    accumulates and the protocol becomes insolvent. Must be runnable
    by any third party.
14. A risk monitoring service that tracks protocol health metrics:
    per-market utilisation, position LTVs approaching liquidation
    thresholds, and oracle feed status. Must expose metrics via an API
    or dashboard.

#### Usability

1. Build the program using the [SPEL framework](https://github.com/logos-co/spel), which
   generates the IDL and client code from the program definition.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations.
4. Liquidator bot and risk monitoring services are implemented as
   Logos modules, accompanied with Logos core headless CLIs/daemons.
5. The mini-app supports creating markets, supplying, borrowing,
   repaying, withdrawing, and viewing position health per market.
6. Position LTV is displayed per market per borrower, queryable
   on-chain and surfaced in both CLI and mini-app.
7. Current supply APY, borrow APR, and utilisation are displayed per
   market in the mini-app and CLI.
8. When interacting via a private account, the SDK must handle the
   atomic deshield (deposit token + native gas) as a single
   indivisible user action, preventing accidental privacy leaks from
   externally funding the intermediate account.
9. When interacting via a private account, before each operation the
   mini-app must show the estimated transaction fee and confirm that
   the user's shielded balance covers both the operation amount and
   fees within the single deshield action. If the balance is
   insufficient, a clear, actionable error must be shown before any
   transaction is submitted, preventing partial deshields that could
   leave funds stranded in an ephemeral account.
10. The mini-app must preview the position LTV impact of a borrow or
    withdrawal before the user confirms: displaying both the current
    LTV and the projected LTV after the operation.

#### Reliability

1. The program rejects borrow and liquidation operations when the
   oracle price feed is older than a configurable staleness threshold.
2. Liquidation triggers use a time-weighted or confidence-adjusted
   price, not raw spot price, to resist single-transaction price
   manipulation.
3. If a market's oracle feed becomes permanently unavailable, the
   program rejects only operations that depend on that feed (borrows,
   liquidations) for the affected market. All other markets and
   operations continue unaffected.
4. Bad debt remaining after a liquidation exhausts available
   collateral is socialised among the market's suppliers (supply
   shares decrease in value). Bad debt in one market does not affect
   any other market.

#### Performance

1. Document the transaction size of each operation (market creation,
   supply, borrow, repay, withdraw, liquidate, flash loan) on LEZ.
   LEZ's block size is limited and this budget may change during
   testnet.
2. The program supports at least 20 independent markets without
   exceeding LEZ compute limits on any single user operation.

#### Supportability

1. The program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
5. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for interacting with
   the program via CLI and mini-app.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→interact→reshield pattern for private
   account interaction. When a user chooses the private account path,
   the SDK must enforce the complete deshield→interact→reshield
   pattern; the reshield step must not be skippable.
2. When using the private account path, the mini-app must display a
   pre-confirmation summary for each operation that clearly identifies
   what will be visible on-chain (amounts, asset type, market address,
   ephemeral intermediary account) and what will remain private (the
   originating private account, the destination of re-shielded tokens,
   and any link between separate interactions by the same user).
3. When using the private account path, the SDK must validate that the
   target account for re-shielding is a private (shielded) account
   before submitting the transaction, and reject the operation with an
   explicit error if it is not.
4. The ephemeral public account (account A) created during the
   deshield step must never be reused across operations. Each protocol
   interaction from a private account must use a freshly generated
   account with no prior on-chain history.

### Soft Requirements

If possible.

#### Reliability
1. Multi-oracle redundancy: at least two independent oracle providers
   per market, with fallback when the primary is stale or unavailable.
2. Formal verification of critical invariants: (a) a position below
   the market's LLTV cannot be liquidated, (b) supply share value is
   monotonically non-decreasing absent bad debt, (c) the IRM's borrow
   rate output is monotonically increasing with utilisation.

### Out of Scope

The following are explicitly excluded from this RFP.

Curation layer features, deferred to [RFP-012](./RFP-012-advanced-lending-features.md):

- Curated vault abstraction: single-asset deposit vaults allocating
  across multiple markets with a curator-managed strategy
- Vault share tokens (receipt tokens): transferable SPL tokens
  representing a vault deposit position
- Supply and borrow caps (enforced at the vault layer, not the core)
- Allocator and guardian roles for vault governance

Other exclusions:

- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Multiple collateral per position (each market has exactly one
  collateral token; multi-collateral strategies are composed at the
  vault layer)

### Privacy Architecture

See [Appendix: Lending Platform Context](../appendix/lending-platform.md#privacy-architecture).

## ⚠ Platform Dependencies

See [Appendix: Lending Platform Context](../appendix/lending-platform.md#platform-dependencies).

## 👤 Recommended Team Profile

Team experienced with:

- DeFi lending protocol design (Morpho Blue, Aave, Compound, Kamino,
  or similar)
- Solana or SVM program development (Anchor or native)
- Interest rate modelling and utilisation curve design
- Liquidation mechanism design and MEV considerations
- Oracle integration and price feed security
- Writing and running on-chain tests (e.g. Bankrun, Anchor tests)
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **16 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-001 — Admin Authority Library](/RFPs/RFP-001-admin-authority-lib.md) — reference pattern for admin-gated operations (F9, F10, F12)
- [RFP-002 — Freeze Authority Library](/RFPs/RFP-002-freeze-authority-lib.md)
- [Appendix: Lending Platform Context](../appendix/lending-platform.md) — shared privacy architecture and platform dependencies
- TODO: LEE official doc
- TODO: Oracle integration guide for LEZ
- TODO: SPEL framework documentation


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
