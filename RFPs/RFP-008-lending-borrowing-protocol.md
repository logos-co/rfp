---
id: RFP-008
title: Lending & Borrowing Protocol
tier: XL
funding: $XXXXX
status: open
dependencies: See Appendix (Platform Dependencies)
category: Applications & Integrations
---


# RFP-008 — Lending & Borrowing Protocol

## 🧭 Overview

Build a permissionless, isolated-market lending protocol on LEZ,
following the Morpho Blue design: each market is an independent lending
pair defined by five immutable parameters (loan token, collateral
token, oracle, interest rate model, liquidation LTV). Anyone can create
a market from enabled parameters; the core protocol is minimal,
non-upgradeable, and governance-free at the market level. The protocol
includes the AdaptiveCurveIRM interest rate model, permissionless
liquidation, oracle-fed price data, and flash loans via the LEZ
tail-call model.

Lending is the capital-efficiency engine of any DeFi ecosystem. It
unlocks idle assets by letting holders earn yield and borrowers access
liquidity without selling. Morpho Blue demonstrated the power of this
minimal, isolated-market approach: its core contract is approximately
600 lines of Solidity, yet the Morpho Blue core accumulated ~$4.9B
TVL across 200+ markets on Ethereum and Base by 2026-04, with the
full Morpho stack (Blue + MetaMorpho vaults) reaching $11.78B TVL
and ~$4B active loans by 2026-05. For the
Logos ecosystem, a lending protocol creates demand for assets deployed
on LEZ, drives TVL, and provides a composability surface that other
programs can build on.

This RFP funds the **software** of a Morpho Blue equivalent on LEZ:
the foundational isolated-market lending primitives (market creation,
supply, borrow, repay, withdraw, liquidation, interest accrual, oracle
integration, flash loans), plus reference implementations of a
liquidator and risk monitor for third parties to fork. Mainnet
deployment, choice of the LLTV grid that production markets can use,
choice of approved oracles and IRMs, the admin authority on mainnet,
and any related governance are explicitly out of scope and will be
handled by Logos through a separate process. The curation layer
(single-asset deposit vaults that allocate across multiple markets,
vault share tokens, supply and borrow caps) is addressed in
[RFP-012](./RFP-012-curated-lending-vaults.md). The applying team
should have experience building or contributing to on-chain lending
protocols and be comfortable with interest rate modelling, liquidation
mechanics, and oracle integration.

## 🔥 Why This Matters

Lending protocols anchor DeFi ecosystems. Aave on Polygon reached over
$1B TVL within months of deployment and was the largest application on
the chain at that time. On Solana, Kamino grew to $2.8B TVL and is the
largest source of borrowed liquidity on the chain.

Without lending, assets on LEZ sit idle. Holders have no way to earn
yield, builders have no liquidity primitive to compose with, and the
ecosystem lacks the capital-efficiency layer needed to attract serious
DeFi activity. A lending protocol is one of the highest-impact
applications for growing the Logos ecosystem.

The Morpho Blue model offers specific advantages for a nascent
ecosystem like LEZ. Permissionless market creation removes governance
bottlenecks for listing new assets, letting the market decide which
pairs are worth lending against. Isolated markets eliminate systemic
contagion: bad debt in one market cannot cascade to others. And the
minimal core (no upgradeable parameters, no complex shared-pool
accounting) is easier to audit, formally verify, and reason about,
which is critical when the protocol is the first lending primitive on a
new chain.

Beyond TVL, lending creates downstream demand: a borrowing market for
stablecoins and the collateral layer that synthetic assets and
structured products are built on.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. The lending protocol is a **single deployed LEZ program** (a
   singleton) that holds all markets in its own state, keyed by market
   id. There is no per-market program deployment. This is what allows
   F14 (flash loans) to span every market in one transaction and what
   keeps cross-market accounting and gas costs predictable.
2. The core program is **immutable and non-upgradeable** once
   deployed. There is no proxy, no migration path, and no admin
   function that can pause the program, change core math, or
   reconfigure a market after creation. Markets, once created, are
   immutable: their five parameters (loan token, collateral token,
   oracle, IRM, LLTV) cannot be changed.
3. Anyone can create a lending market by specifying the five
   parameters above. The market id is the deterministic hash of those
   five parameters. Market creation is permissionless; no admin
   approval is required. The oracle program is chosen by the market
   creator and is not gated by the admin authority: the trust model
   assumes lenders evaluate the oracle before supplying. The mini-app
   must surface the oracle program ID per market so lenders can verify
   it (related to U6, U7).
4. Users can supply the loan token to a specific market. The protocol
   tracks supply shares internally per market per account (no receipt
   token at the core level). Supply shares represent a proportional
   claim on the market's loan token balance plus accrued interest.
5. Users can withdraw supplied loan tokens from a market at any time,
   subject to available liquidity in that market.
6. Users can deposit the market's collateral token and borrow the loan
   token, up to the market's LLTV ratio.
7. Users can repay borrowed loan tokens in full or partially.
8. The launch IRM is **AdaptiveCurveIRM**: an autonomous,
   utilisation-targeting interest rate model with a ~90% target
   utilisation and cumulative time-decay adjustment, requiring no
   per-asset retuning. The IRM is itself a separate, immutable LEZ
   program referenced by each market at creation. Reference:
   [Morpho: Introducing AdaptiveCurveIRM](https://morpho.org/blog/introducing-the-adaptivecurveirm-efficient-and-autonomous/),
   [Morpho docs: IRM](https://docs.morpho.org/get-started/resources/contracts/irm/).
   The IRM program exposes pure read functions that return supply APY
   and borrow APR for a given market id; no state mutation is required
   to query rates.
9. Interest accrual is lazy (computed on interaction), not via a
   separate crank transaction per block.
10. When a borrower's position LTV exceeds the market's LLTV, any
    account can permissionlessly liquidate the position by repaying a
    portion of the debt and receiving equivalent collateral plus a
    liquidation incentive. The liquidation incentive factor (LIF) is
    derived from the market's LLTV via the Morpho Blue formula:
    `LIF = min(M, 1 / (β·LLTV + (1−β)))` with cursor β = 0.3 and
    maximum bound M = 1.15. Higher LLTV markets yield smaller
    incentives; lower LLTV markets yield larger incentives, capped at
    M. Reference:
    [Morpho docs: Liquidation](https://docs.morpho.org/morpho/concepts/liquidation/).
    Applicants may deviate from these parameters only with explicit
    justification; the formula shape (LIF derived from LLTV with a
    cursor and a max bound) is fixed.
11. **Admin scope and non-scope.** An admin authority manages a
    bounded set of protocol-level functions:
    (a) enabling LLTV values that market creators can select from,
    (b) enabling IRM implementations that market creators can select
    from (AdaptiveCurveIRM is the only IRM at launch),
    (c) setting the optional protocol fee percentage and fee
    recipient.
    The admin **cannot** pause the program, upgrade or migrate the
    core, modify any deployed market, change market parameters after
    creation, or seize user funds. On testnet, the admin function for
    enabling LLTV values must allow adding and removing values to
    support iteration; on mainnet, the choice of LLTV grid and the
    governance around the admin authority are the responsibility of
    the Logos deployment process and are not in this RFP's scope.
12. An optional protocol fee diverts a configurable fraction of
    interest accrued in each market to a fee recipient account. Both
    the percentage and the recipient are set by the admin authority.
13. Each market references the oracle program specified at creation
    for collateral valuation and liquidation triggers. The oracle is
    a separate LEZ program.
14. **Flash loans (LEZ-native).** A user can borrow assets from any
    market within a single transaction. The protocol's flash-loan
    entrypoint transfers the requested loan tokens to the borrower
    and tail-calls a borrower-supplied continuation, handing it an
    **unforgeable repayment capability** (per LP-0015's
    capability-protected continuation model). The borrower's
    continuation must, before the transaction terminates, tail-call
    back into the lending program's internal `repay_flash`
    continuation with the capability and the principal plus fee. If
    the transaction does not complete the chain with full repayment,
    the entire transaction reverts atomically and no funds leave the
    market. There is no synchronous reentrancy: control hands off
    fully at each tail call, so the EVM concepts of CEI ordering and
    reentrancy locks do not apply. Flash loans are **zero-fee**,
    matching Morpho Blue's deployed implementation (the deployed
    `Morpho.sol` collects exactly `assets` on repayment with no
    premium). Repayment must equal the principal exactly; no fee is
    charged.
15. **Reference liquidator.** Ship a reference liquidator daemon
    that continuously monitors all markets and executes liquidations
    when positions exceed their market's LLTV. This is provided as
    example software for third parties to fork and operate; Logos
    does not commit to running it. Liquidation on-chain remains
    fully permissionless: anyone can write and run their own.
16. **Reference risk monitor.** Ship a reference risk-monitoring
    service that tracks protocol health metrics: per-market
    utilisation, position LTVs approaching liquidation thresholds,
    and oracle feed status. Like the liquidator, this is reference
    software for third parties; it must expose metrics via an API or
    dashboard.

#### Usability

1. Build the program using the [SPEL framework](https://github.com/logos-co/spel), which
   generates the IDL and client code from the program definition.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations.
4. The reference liquidator (F15) and reference risk monitor (F16)
   are implemented as Logos modules with headless CLIs/daemons,
   suitable for third parties to fork and run independently.
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
5. The following invariants are **formally verified** before mainnet
   release (Certora-style or equivalent), as a precondition of the
   core program being immutable: (a) a position whose LTV is below
   its market's LLTV cannot be liquidated, (b) absent socialised bad
   debt, supply share value is monotonically non-decreasing,
   (c) the AdaptiveCurveIRM borrow rate is monotonically non-
   decreasing in market utilisation, (d) the protocol cannot enter a
   state where total supplied is less than total borrowed plus on-
   chain liquid balance (solvency).

#### Performance

1. Document the transaction size of each operation (market creation,
   supply, borrow, repay, withdraw, liquidate, flash loan) on LEZ.
   LEZ's block size is limited and this budget may change during
   testnet.
2. The program scales linearly in the number of markets without
   per-market state polluting per-user-operation compute. The actual
   per-deployment market count is bounded by LEZ's per-transaction
   compute budget (see Platform Dependencies, Risks, Compute budget,
   for the benchmark deliverable). As a reference order of magnitude,
   Morpho Blue runs 200+ active markets on Ethereum and Base
   (2026-04); LEZ's per-tx compute envelope will determine how close
   the singleton can run to that figure. The applicant must report
   the maximum supported market count after benchmarking, not commit
   to a fixed figure at proposal time.

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
6. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for supply,
   borrow, repay, withdraw, and liquidation.
7. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
8. Provide Figma designs or equivalent for the mini-app GUI.
9. Provide a privacy and anonymisation properties document covering:
   what on-chain state and transaction data is visible to observers;
   what data is protected when the private account path is used; trust
   assumptions, specifying which guarantees are enforced by the on-chain
   program and which depend on correct client behaviour; and what
   happens if a user bypasses the expected interaction path. See the
   [Privacy Architecture](#privacy-architecture) section below for the
   baseline this document must align with.
10. **Audit programme.** The proposal must include a planned audit
    programme covering the lending program, AdaptiveCurveIRM,
    bundler (if delivered as a soft requirement), reference
    liquidator, and reference risk monitor. The proposal must name
    at least one tier-1 audit firm the applicant intends to engage
    (for example: OpenZeppelin, Trail of Bits, Spearbit, Cantina,
    ChainSecurity, Certora, Halborn), include the audit budget as a
    line item in the proposal, and include the audit timeline ahead
    of any mainnet recommendation. A second independent audit is
    strongly preferred for the lending program given immutability.
    Audit reports must be published with the codebase before mainnet
    deployment is recommended.

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
   account with no prior on-chain history. Because LEZ accounts are
   keypair-derived (Solana-style), generating an account with no
   on-chain history is trivial; preventing reuse, however, is an SDK
   responsibility. Applicants must document how the SDK enforces
   single-use (for example, fresh-keypair-per-operation, a
   deterministic single-use derivation scheme tied to a nonce, or a
   local registry of consumed ephemeral keys) and how it survives
   client restarts and multi-device usage without reuse.

### Soft Requirements

If possible.

#### Reliability
1. Multi-oracle redundancy: at least two independent oracle providers
   per market, with fallback when the primary is stale or unavailable.

#### Functionality
1. On-chain bundler program: a separate LEZ program that composes
   approve + supply + borrow (and similar multi-step user actions)
   into one atomic tail-call chain, modelled on Morpho's
   [`morpho-blue-bundlers`](https://github.com/morpho-org/morpho-blue-bundlers).
   The SDK uses the bundler rather than re-implementing composition
   off-chain.

### Out of Scope

The following are explicitly excluded from this RFP.

This RFP funds **software**. The mainnet deployment of that software,
and the governance around it, is handled by Logos through a separate
process. The following are therefore out of scope:

- Mainnet deployment of the lending program.
- The choice of the LLTV grid that production markets can use.
- The choice of approved oracle programs and IRMs on mainnet.
- Holder, custody, or rotation policy for the admin authority on
  mainnet.
- Any governance design (token, DAO, multisig) around the admin
  authority.

Curation layer features, deferred to [RFP-012](./RFP-012-curated-lending-vaults.md):

- Curated vault abstraction: single-asset deposit vaults allocating
  across multiple markets with a curator-managed strategy
- Vault share tokens (receipt tokens): transferable LEZ fungible
  tokens representing a vault deposit position
- Supply and borrow caps (enforced at the vault layer, not the core)
- Allocator and sentinel roles for vault governance

Other exclusions:

- Alternative IRM implementations beyond AdaptiveCurveIRM. Additional
  IRMs may be enabled later by the admin, but are not deliverables of
  this RFP.
- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Each market has exactly one collateral token; users compose
  multi-collateral exposure by holding positions across multiple
  markets, optionally via curated vaults (RFP-012).

### Open Questions

> **[OPEN] Off-chain indexing surface (indexer + GraphQL API).**
> Morpho ships a Messari-schema subgraph stack and a hosted GraphQL
> API at `api.morpho.org` alongside the contracts. Frontends, the
> Morpho SDK, vault allocators, and risk dashboards all read from
> this surface rather than from chain directly. On LEZ this work has
> not been scoped or owned: it could sit inside this RFP (single
> team ships program + indexer), inside RFP-012 (the vault layer
> takes a hard dependency on indexing it ships itself), or as a
> separate RFP funded independently. The research vault has only a
> shallow note on Morpho's stack so far ([`research-lending/projects/morpho-blue.md`](https://github.com/marclawclaw/research-lending/blob/master/projects/morpho-blue.md)),
> not enough to choose. Feedback from reviewers on where this work
> should land is welcome.

### Privacy Architecture

The protocol state (lending markets, positions, interest indices, and
token accounts) lives in **public (on-chain) state**. This is a
deliberate architectural choice: public state enables permissionless
liquidation, oracle integration, and composability without open
cryptographic research challenges.

User privacy is optionally enforced at the UX layer. The mini-app and
SDK support both direct public account interaction and private account
interaction via the deshield, interact, reshield pattern. When users
opt to interact from a private account, the SDK must enforce the
complete pattern as described below.

#### Interaction flow

For every protocol operation (supply, borrow, repay, withdraw):

1. The user initiates the action from their private account. The SDK
   deshields to a **fresh, single-use** public account (account A)
   with no prior on-chain history. The deshield atomically transfers
   both the operation token **and** enough native token for gas in a
   single indivisible action.
2. Account A executes the protocol operation (e.g. supplies assets to
   a market).
3. Account A shields any outputs (withdrawn assets, repaid collateral)
   back to the user's private account. Account A is never reused.

> **Gas:** Both the operation token and gas must come exclusively from
> the deshield in step 1. Funding account A from any external source
> (such as a CEX withdrawal or a known wallet) creates an on-chain
> link to an existing identity and breaks the privacy guarantee. The
> SDK must make this impossible; the atomic deshield is a single,
> indivisible user action.

#### What is public (observable on-chain)

- All market state: asset pairs, interest rate parameters, total
  supplied, total borrowed, utilisation, supply APY, borrow APR.
- All positions: collateral amounts, debt amounts, position LTV,
  attributed to the ephemeral intermediary account, not the
  private user.
- All protocol operations and their amounts.
- Oracle price feeds and liquidation events.

#### What is private

- Which private account funded any supply or borrow.
- Where withdrawn assets or other outputs go after re-shielding.
- Any link between multiple protocol interactions by the same user
  (no on-chain linkability across ephemeral accounts).

## ⚠ Platform Dependencies

This RFP is open for proposal submission. However, development is
blocked until the dependencies below are resolved. LEZ has similar
programming capabilities to Solana but several primitives required by
a lending protocol are not yet available.

### Hard blockers

These must be available on LEZ before this RFP can open.

#### Oracle provider

No oracle provider is available on LEZ. The lending protocol requires
external price feeds for collateral valuation and liquidation triggers.

#### On-chain clock / timestamp

Interest accrual is Δt-driven and computed lazily on interaction
(F8, F9). Without a reliable on-chain timestamp accessible to the
AdaptiveCurveIRM and to the lending program, interest cannot be
computed and the protocol cannot function. RFP-013 already lists
this as a hard blocker; the same dependency applies here.

#### General cross-program calls (LP-0015)

LEZ uses a tail-call execution model rather than Solana's CPI
(Cross-Program Invocation). In Solana's model, a program can call
another program mid-execution and resume when the call returns. In
LEZ's model, a tail call hands off control entirely; there is no
return.

A lending operation like "supply" needs to: (1) call the token
program to transfer assets into the market, then (2) continue
executing to update interest indices and write position state. Without
general cross-program calls, step 2 cannot happen after step 1. Each
continuation would need to be a separate externally callable
entrypoint, which is fragile and insecure (anyone could call the
continuation directly, bypassing the token transfer).

[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this by introducing
internal-only entrypoints protected by an unforgeable capability, so
the lending program can tail-call the token program and have control
return to a protected continuation. F14 (flash loans) depends directly
on this capability mechanism. This prize is currently **open**.

#### Token authorities (LP-0013)

The lending program is a token custodian: it holds supplied loan
tokens per market, holds collateral on behalf of borrowers, pays
liquidators atomically from collateral, and routes the protocol fee.
This requires the transfer-authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md),
currently **open**.

#### Event emission (LP-0012)

The reference liquidator (F15) and reference risk monitor (F16) need
to observe on-chain state changes efficiently. Without structured
events, both services must poll all market and position accounts,
which is expensive and unreliable, and any third-party liquidator
fork will hit the same problem. Insolvency is contained by timely
liquidations; timely liquidations are gated by efficient indexing.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

### Risks

#### Compute budget

LEZ currently processes one private transaction per block (as of
2026-04). Public LEE documentation that pins this down is still
pending; once it lands, link it from the Resources section. Liquidation
is the most compute-intensive lending operation: it reads the
borrower's position, collateral balance, oracle price, interest
indices, and market parameters, then writes updated state. On Solana,
where the per-instruction default is 200,000 compute units and the
per-transaction maximum is 1.4M compute units (see
[Solana compute budget](https://solana.com/docs/core/fees/compute-budget)),
a liquidation typically consumes hundreds of thousands of compute
units; the exact figure for LEZ depends on the implementation. If
LEZ's per-transaction compute budget is insufficient, liquidations
will fail and the protocol risks insolvency.

**Deliverable.** Applicants must benchmark the compute cost of
liquidation on LEZ devnet/testnet once the protocol is integrated and
include the figures in the README. If liquidation does not fit within
the per-transaction budget, applicants must propose a remediation
(for example, a two-phase liquidation via tail-call continuation) and
implement it within the funded scope.

## 👤 Recommended Team Profile

Team experienced with:

- Rust program development on LEZ (or SPEL / Solana / SVM, which
  transfer directly)
- DeFi lending protocol design (Morpho Blue, Aave, Compound, Kamino,
  or similar)
- Interest rate modelling and utilisation curve design
  (AdaptiveCurveIRM in particular)
- Liquidation mechanism design and MEV considerations
- Oracle integration and price feed security
- Formal verification of smart contracts (Certora or equivalent)
- Writing and running on-chain tests against a LEZ sequencer (or
  equivalent local-validator workflows from Solana, e.g. Bankrun,
  Anchor tests)
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **16 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md): reference pattern for the bounded admin authority (F11 admin scope, F12 protocol fee, F14 flash-loan fee)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- [Appendix: Lending and Borrowing Ecosystem](../appendix/lending-ecosystem.md): ecosystem survey across Ethereum, Solana, BTCFi, and long-tail chains; Morpho Blue deep dive
- TODO: LEE official doc
- TODO: Oracle integration guide for LEZ
- TODO: SPEL framework documentation


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
