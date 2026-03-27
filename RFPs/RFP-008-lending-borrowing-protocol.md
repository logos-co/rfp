---
id: RFP-008
title: Lending & Borrowing Protocol
tier: XL
funding: $XXXXX
status: draft
dependencies: See Platform Dependencies section
category: Applications & Integrations
---


# RFP-008 — Lending & Borrowing Protocol

## 🧭 Overview

Build a lending and borrowing protocol on LEZ that allows users to supply
assets to liquidity pools, earn interest, and borrow against
over-collateralised positions. The protocol must include algorithmic
interest rates, a permissionless liquidation engine, oracle-fed price
data, and composable receipt tokens — the core primitives that every
major lending protocol (Aave, Compound, Morpho, Kamino, etc.) depends on.

Lending is the capital-efficiency engine of any DeFi ecosystem. It unlocks
idle assets by letting holders earn yield and borrowers access liquidity
without selling. For the Logos ecosystem, a lending protocol creates
demand for assets deployed on LEZ, drives TVL, and provides a
composability surface that other programs can build on.

This RFP targets Aave v2 core equivalence — the foundational lending
primitives (supply, borrow, repay, withdraw, liquidation, interest
accrual, oracle integration). Advanced features such as eMode, flash
loans, isolated markets, and multiple collateral per position (Aave v3
territory) are out of scope here and addressed in
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

Beyond TVL, lending creates downstream demand: it establishes a
borrowing market for stablecoins, generates liquidation volume for DEXs,
and provides the collateral infrastructure needed for synthetic assets
and structured products. Every major DeFi ecosystem was built on this
foundation.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Users can supply supported assets to liquidity pools and receive
   interest-bearing receipt tokens representing their pool share.
   Receipt tokens must be standard SPL tokens, freely transferable
   and composable with other LEZ programs.
2. Users can borrow supported assets against deposited collateral.
   All borrow positions must be over-collateralised, governed by
   per-asset LTV ratios.
3. Users can repay borrowed assets in full or partially.
4. Users can withdraw supplied assets at any time, subject to
   available pool liquidity.
5. Interest rates are determined by a utilisation-based model that
   increases rates as pool utilisation rises, incentivising repayment
   and maintaining withdrawal liquidity. The specific model design
   is left to the implementer. Parameters must be configurable per
   asset.
6. Interest accrues continuously. Supply APY and borrow APR are
   deterministic functions of pool utilisation, queryable on-chain.
7. When a borrower's health factor (the ratio of weighted collateral
   value to outstanding debt; HF ≥ 1 means solvent, HF < 1 means
   eligible for liquidation) drops below 1, any account can
   permissionlessly liquidate the position by repaying a portion of
   the debt and receiving equivalent collateral at a discount.
   Liquidation is partial (configurable close factor per transaction).
8. Each asset has independently configurable risk parameters: LTV,
   liquidation threshold, liquidation bonus, reserve factor, and
   supply/borrow caps.
9. A reserve factor diverts a percentage of borrow interest into
   protocol reserves as a buffer against bad debt.
10. Price feeds from at least one oracle provider are integrated for
    collateral valuation and liquidation triggers.
11. Per-asset supply and borrow caps are enforced and adjustable
    without program upgrade.
12. A liquidator bot that continuously monitors all borrower
    positions and executes liquidations when health factors drop
    below 1. The bot is the protocol's solvency mechanism — without
    it, bad debt accumulates and the protocol becomes insolvent.
    Must be runnable by any third party.
13. A risk monitoring service that tracks protocol health metrics:
    per-asset utilisation, aggregate collateral ratios, positions
    approaching liquidation thresholds, and oracle feed status.
    Must expose metrics via an API or dashboard.

#### Usability

1. Build the program using the [SPEL framework](https://github.com/logos-co/spel), which
   generates the IDL and client code from the program definition.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations.
4. Liquidator bot and risk monitoring services are implemented as Logos modules, accompanied with Logos core headless CLIs/daemons.
5. The mini-app supports depositing, borrowing, repaying,
   withdrawing, and viewing position health.
6. A single health factor metric is displayed per borrower position,
   queryable on-chain and surfaced in both CLI and mini-app.
7. Current supply APY, borrow APR, and pool utilisation are displayed
   per asset in the mini-app and CLI.
8. When interacting via a private account, the SDK must handle the
   atomic deshield (deposit token + native gas) as a single indivisible
   user action, preventing accidental privacy leaks from externally
   funding the intermediate account.
9. When interacting via a private account, before each operation the
   mini-app must show the estimated transaction fee and confirm that the
   user's shielded balance covers both the operation amount and fees
   within the single deshield action. If the balance is insufficient, a
   clear, actionable error must be shown before any transaction is
   submitted — preventing partial deshields that could leave funds
   stranded in an ephemeral account.
10. The mini-app must preview the health factor impact of a borrow or
    withdrawal before the user confirms: displaying both the current
    health factor and the projected health factor after the operation.

#### Reliability

1. The program rejects borrow and liquidation operations when the
   oracle price feed is older than a configurable staleness threshold.
2. Liquidation triggers use a time-weighted or confidence-adjusted
   price, not raw spot price, to resist single-transaction price
   manipulation.
3. If an asset's oracle feed becomes permanently unavailable, the
   program rejects only operations that depend on that feed (borrows,
   liquidations) for the affected asset. All other assets and
   operations continue unaffected.
4. Bad debt remaining after a liquidation exhausts available
   collateral is tracked and socialised across the reserve fund.

#### Performance

1. Document the transaction size of each operation (supply, borrow,
   repay, withdraw, liquidate) on LEZ. LEZ's block size is limited
   and this budget may change during testnet.
2. Interest accrual is lazy (computed on interaction), not via a
   separate crank transaction per block.
3. The program supports at least 5 distinct asset markets
   simultaneously without exceeding LEZ compute limits on any
   single user operation.

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
   pattern — the reshield step must not be skippable.
2. When using the private account path, the mini-app must display a
   pre-confirmation summary for each operation that clearly identifies
   what will be visible on-chain (amounts, asset type, pool address,
   ephemeral intermediary account) and what will remain private (the
   originating private account, the destination of re-shielded tokens,
   and any link between separate interactions by the same user).
3. When using the private account path, the SDK must validate that the
   target receipt token account for re-shielding is a private
   (shielded) account before submitting the transaction, and reject the
   operation with an explicit error if it is not.
4. The ephemeral public account (account A) created during the deshield
   step must never be reused across operations. Each protocol
   interaction from a private account must use a freshly generated
   account with no prior on-chain history.

### Soft Requirements

If possible.

#### Reliability
1. Multi-oracle redundancy: at least two independent oracle providers,
   with fallback when the primary is stale or unavailable.
2. Formal verification of critical invariants: (a) a healthy position
   cannot be liquidated, (b) receipt token supply equals total
   supplied assets plus accrued interest, (c) interest rate output is
   monotonically increasing with utilisation.

### Out of Scope

The following are explicitly excluded from this RFP.

Advanced lending features — deferred to [RFP-012](./RFP-012-advanced-lending-features.md):

- eMode (efficiency mode): correlated asset groups with elevated LTV ratios
- Isolated markets: higher-risk assets in isolated pools with independent debt ceilings
- Flash loans: uncollateralised single-transaction loans
- Multiple collateral per position: aggregate borrowing power from a weighted collateral sum
- Curated vault abstraction: single-asset deposit vaults allocating across multiple markets
- All risk parameters adjustable without program upgrade

Other exclusions:

- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products

### Privacy Architecture

The protocol state — lending pools, positions, interest indices, and
receipt token accounts — lives in **public (on-chain) state**. This
is a deliberate architectural choice: public state enables
permissionless liquidation, oracle integration, and composability
without open cryptographic research challenges.

User privacy is optionally enforced at the UX layer. The mini-app and
SDK support both direct public account interaction and private account
interaction via the deshield→interact→reshield pattern. When users opt
to interact from a private account, the SDK must enforce the complete
pattern as described below.

#### Interaction flow

For every protocol operation (supply, borrow, repay, withdraw):

1. The user initiates the action from their private account. The SDK
   deshields to a **fresh, single-use** public account (account A)
   with no prior on-chain history. The deshield atomically transfers
   both the operation token **and** enough native token for gas in a
   single indivisible action.
2. Account A executes the protocol operation (e.g. supplies assets,
   receives receipt tokens).
3. Account A shields any outputs (receipt tokens, withdrawn assets)
   back to the user's private account. Account A is never reused.

> **Gas:** Both the operation token and gas must come exclusively from
> the deshield in step 1. Funding account A from any external source
> — such as a CEX withdrawal or a known wallet — creates an on-chain
> link to an existing identity and breaks the privacy guarantee. The
> SDK must make this impossible; the atomic deshield is a single,
> indivisible user action.

#### What is public (observable on-chain)

- All pool state: asset type, interest rate parameters, total
  supplied, total borrowed, utilisation, supply APY, borrow APR.
- All positions: collateral amounts, debt amounts, health factor
  — attributed to the ephemeral intermediary account, not the
  private user.
- All protocol operations and their amounts.
- Oracle price feeds and liquidation events.

#### What is private

- Which private account funded any supply or borrow.
- Where receipt tokens or withdrawn assets go after re-shielding.
- Any link between multiple protocol interactions by the same user
  (no on-chain linkability across ephemeral accounts).

## ⚠ Platform Dependencies

This RFP remains in **draft** until the dependencies below are resolved.
LEZ has similar programming capabilities to Solana but several
primitives required by a lending protocol are not yet available.

### Hard blockers

These must be available on LEZ before this RFP can open.

#### Oracle provider

No oracle provider is available on LEZ. The lending protocol requires
external price feeds for collateral valuation and liquidation triggers.
Every hard requirement related to liquidation (F7), oracle integration
(F10), and reliability (R1–R3) depends on this.

#### On-chain clock / timestamp

LEZ does not yet have on-chain block time. Interest accrual — the
core economic mechanic of a lending protocol — requires knowing how
much time has elapsed between interactions. Without a reliable on-chain
timestamp, interest rates, supply APY, and borrow APR cannot be
computed.

#### General cross-program calls (LP-0015)

LEZ uses a tail-call execution model rather than Solana's CPI
(Cross-Program Invocation). In Solana's model, a program can call
another program mid-execution and resume when the call returns. In
LEZ's model, a tail call hands off control entirely — there is no
return.

A lending operation like "supply" needs to: (1) call the token
program to transfer assets into the pool, then (2) continue executing
to update interest indices, mint receipt tokens, and write position
state. Without general cross-program calls, step 2 cannot happen after
step 1. Each continuation would need to be a separate externally
callable entrypoint, which is fragile and insecure (anyone could call
the continuation directly, bypassing the token transfer).

[LP-0015](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this by introducing
internal-only entrypoints protected by an unforgeable capability, so
the lending program can tail-call the token program and have control
return to a protected continuation. This prize is currently **open**.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission (LP-0012)

Liquidator bots and indexers need to monitor positions and react to
on-chain state changes. Without structured events, off-chain services
must poll all accounts, which is expensive and unreliable.

[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

### Risks

#### Compute budget

LEZ currently processes one private transaction per block. Liquidation
is the most compute-intensive lending operation: it reads the
borrower's position, multiple collateral balances, multiple oracle
prices, interest indices, and risk parameters, then writes updated
state. On Solana, a liquidation can consume 200–400K compute units.
If LEZ's per-transaction compute budget is insufficient, liquidations
will fail and the protocol risks insolvency. This needs benchmarking
once the protocol is under development.

## 👤 Recommended Team Profile

Team experienced with:

- DeFi lending protocol design (Aave, Compound, Kamino, Morpho, or
  similar)
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

- [RFP-001 — Admin Authority](/RFPs/RFP-001-admin-authority-poc.md)
- [RFP-002 — Freeze Authority](/RFPs/RFP-002-freeze-authority-poc.md)
- TODO: LEE official doc
- TODO: Oracle integration guide for LEZ
- TODO: SPEL framework documentation


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
