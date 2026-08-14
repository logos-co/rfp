---
id: RFP-008
title: Lending & Borrowing Protocol
tier: XL
status: closed
category: Applications & Integrations
dependencies:
  - id: LP-0013
    reason: Token transfer-authority primitives are required for the lending program to custody supplied assets and collateral and to pay liquidators atomically.
  - id: RFP-020
    reason: At least one oracle provider must exist on LEZ for collateral valuation and liquidation triggers (F13); RFP-020 delivers external price feeds (RFP-019 TWAP is an alternative once a DEX is live).
  - id: LP-0015
    reason: General cross-program calls via tail calls, used to compose token transfers with subsequent state updates and required by the flash loan model (F14).
  - id: LP-0012
    reason: Structured event emission, used by the reference liquidator and risk monitor to observe on-chain state changes.
---

# RFP-008 — Lending & Borrowing Protocol

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) for complying with any legal and regulatory requirements that apply
> to them. Software developed under the Program is published and maintained by
> its developers, not by Logos.
>
> Anyone who chooses to deploy, host, operate or use software developed under
> the Program, whether or not they were awarded a grant under the Program, does
> so at their own risk and is solely responsible for complying with any legal or
> regulatory requirements that apply to them. See the
> [Terms & Conditions](../TERMS_AND_CONDITIONS.md).
>
> Deploying the software described in this RFP, operating any service based on
> it, or carrying on business through it may amount to regulated activity in
> some jurisdictions, including where it involves holding or managing users'
> assets or providing services to others. Whoever conducts any such activity
> does so as principal, in their own name, and is solely responsible for
> assessing its regulatory treatment, including any licensing, registration,
> sanctions or anti-money laundering obligations that may apply to them. Logos
> does not make any representation, provides any advice or assumes any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

Build a permissionless, isolated-market lending protocol on LEZ, following the
Morpho Blue design: each market is an independent lending pair defined by five
immutable parameters (loan token, collateral token, oracle, interest rate model,
liquidation LTV). Anyone can create a market from enabled parameters; the core
protocol is minimal, non-upgradeable, and governance-free at the market level.
The protocol includes the AdaptiveCurveIRM interest rate model, permissionless
liquidation, oracle-fed price data, and flash loans via the LEZ tail-call model.

Lending is the capital-efficiency engine of any DeFi ecosystem. It unlocks idle
assets by letting holders earn yield and borrowers access liquidity without
selling. Morpho Blue demonstrated the power of this minimal, isolated-market
approach: its core contract is approximately 650 lines of Solidity, yet the
Morpho Blue core accumulated ~$7.2B TVL across 200+ markets on Ethereum and Base
by 2026-04, with the full Morpho stack (Blue + MetaMorpho vaults) reaching
$11.78B TVL and ~$4B active loans by 2026-05. For the Logos ecosystem, a lending
protocol creates demand for assets deployed on LEZ, drives TVL, and provides a
composability surface that other programs can build on.

This RFP funds the **software** of a Morpho Blue equivalent on LEZ: the
foundational isolated-market lending primitives (market creation, supply,
borrow, repay, withdraw, liquidation, interest accrual, oracle integration,
flash loans), plus reference implementations of a liquidator and risk monitor
for third parties to fork. Mainnet deployment, choice of the LLTV grid that
production markets can use, choice of admin-whitelisted IRMs on mainnet, the
admin authority on mainnet, and any related governance are explicitly out of
scope and will be handled through a separate process. The curation layer
(single-asset deposit vaults that allocate across multiple markets, vault share
tokens, supply and borrow caps) is addressed in
[RFP-012](./RFP-012-curated-lending-vaults.md). The applying team should have
experience building or contributing to on-chain lending protocols and be
comfortable with interest rate modelling, liquidation mechanics, and oracle
integration.

## 🔥 Why This Matters

Lending protocols anchor DeFi ecosystems. Aave on Polygon reached over $1B TVL
within months of deployment and was the largest application on the chain at that
time. On Solana, Kamino grew to $2.8B TVL and is the largest source of borrowed
liquidity on the chain.

Without lending, assets on LEZ sit idle. Holders have no way to earn yield,
builders have no liquidity primitive to compose with, and the ecosystem lacks
the capital-efficiency layer needed to attract serious DeFi activity. A lending
protocol is one of the highest-impact applications for growing the Logos
ecosystem.

The Morpho Blue model offers specific advantages for a nascent ecosystem like
LEZ. Permissionless market creation removes governance bottlenecks for listing
new assets, letting the market decide which pairs are worth lending against.
Isolated markets eliminate systemic contagion: bad debt in one market cannot
cascade to others. And the minimal core (no upgradeable parameters, no complex
shared-pool accounting) is easier to audit, formally verify, and reason about,
which is critical when the protocol is the first lending primitive on a new
chain.

Beyond TVL, lending creates downstream demand: a borrowing market for
stablecoins and the collateral layer that synthetic assets and structured
products are built on.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

01. The lending protocol is a **single deployed LEZ program** (a singleton) that
    holds all markets in its own state, keyed by market id. There is no
    per-market program deployment. This is what allows F14 (flash loans) to span
    every market in one transaction and what keeps cross-market accounting and
    gas costs predictable.

02. The core program is **immutable and non-upgradeable** once deployed. There
    is no proxy, no migration path, and no admin function that can pause the
    program, change core math, or reconfigure a market after creation. Markets,
    once created, are immutable: their five parameters (loan token, collateral
    token, oracle, IRM, LLTV) cannot be changed.

03. Anyone can create a lending market by specifying the five parameters above.
    The market id is the deterministic hash of those five parameters. Market
    creation is permissionless; no admin approval is required. The oracle
    program is chosen by the market creator and is not gated by the admin
    authority: the trust model assumes lenders evaluate the oracle before
    supplying. The GUI must surface the oracle program ID per market so lenders
    can verify it (related to U7, U8).

04. Users can supply the loan token to a specific market. The protocol tracks
    supply shares internally per market per account (no receipt token at the
    core level). Supply shares represent a proportional claim on the market's
    loan token balance plus accrued interest.

05. Users can withdraw supplied loan tokens from a market at any time, subject
    to available liquidity in that market.

06. Users can deposit the market's collateral token and borrow the loan token,
    up to the market's LLTV ratio.

07. Users can repay borrowed loan tokens in full or partially.

08. The launch IRM is **AdaptiveCurveIRM**: an autonomous, utilisation-targeting
    interest rate model with a ~90% target utilisation and cumulative time-decay
    adjustment, requiring no per-asset retuning. The IRM is itself a separate,
    immutable LEZ program referenced by each market at creation. Reference:
    [Morpho: Introducing AdaptiveCurveIRM](https://morpho.org/blog/introducing-the-adaptivecurveirm-efficient-and-autonomous/),
    [Morpho docs: IRM](https://docs.morpho.org/get-started/resources/contracts/irm/).
    The IRM program exposes pure read functions that return supply APY and
    borrow APR for a given market id; no state mutation is required to query
    rates.

09. Interest accrual is lazy (computed on interaction), not via a separate crank
    transaction per block.

10. When a borrower's position LTV exceeds the market's LLTV, any account can
    permissionlessly liquidate the position by repaying debt and receiving
    equivalent collateral plus a liquidation incentive. There is **no close
    factor**: a liquidator may repay any amount up to the borrower's full debt
    in a single transaction, matching Morpho Blue (which has no close factor on
    the standard liquidation path). A liquidator specifies **either** the
    collateral amount to seize **or** the debt amount to repay (exactly one);
    the protocol derives the other so that the seized collateral value equals
    the repaid debt value multiplied by the liquidation incentive factor (LIF).
    The liquidator's profit is therefore `(LIF − 1) × repaid debt`, paid out of
    the borrower's collateral. The liquidation incentive factor (LIF) is derived
    from the market's LLTV via the Morpho Blue formula:
    `LIF = min(M, 1 / (β·LLTV + (1−β)))` with cursor β = 0.3 and maximum bound M
    = 1.15. Higher LLTV markets yield smaller incentives; lower LLTV markets
    yield larger incentives, capped at M. Reference:
    [Morpho docs: Liquidation](https://docs.morpho.org/morpho/concepts/liquidation/).
    Applicants may deviate from these parameters only with explicit
    justification; the formula shape (LIF derived from LLTV with a cursor and a
    max bound) is fixed.

11. **Admin scope and non-scope.** An admin authority manages a bounded set of
    protocol-level functions: (a) enabling LLTV values that market creators can
    select from, (b) enabling IRM implementations that market creators can
    select from (AdaptiveCurveIRM is the only IRM at launch), (c) setting the
    optional protocol fee percentage and fee recipient. The admin **cannot**
    pause the program, upgrade or migrate the core, modify any deployed market,
    change market parameters after creation, or seize user funds. On testnet,
    the admin function for enabling LLTV values must allow adding and removing
    values to support iteration; on mainnet, the choice of LLTV grid and the
    governance around the admin authority are the responsibility of the Logos
    deployment process and are not in this RFP's scope.

    **No emergency pause is deliberate.** This inherits the Morpho Blue trust
    model: the core is a credibly neutral, immutable primitive that cannot
    freeze user funds or halt liquidations under any condition (including a
    misbehaving oracle, a buggy IRM, or a market with unanticipated risk).
    Pausing a market with active borrows would strand collateral and prevent
    liquidations, which is itself a vector for loss; isolating risk per market
    (F3) and shifting the trust decision to the lender's choice of oracle and
    LLTV at supply time is the substitute for a global pause. Risk containment
    for the protocol as a whole therefore relies on: (a) market isolation, so a
    stale-oracle or misbehaving-asset failure is bounded to lenders of that
    market; (b) the curated vault layer (RFP-012), where the sentinel role can
    revoke a market's cap and stop new allocation flow without touching the
    core; and (c) the reference risk monitor (F16), which surfaces
    oracle-staleness and unhealthy-market signals so vault curators and lenders
    can act. Applicants who believe a different trade-off is warranted (e.g. a
    narrow oracle-halt flag scoped to a single market) must justify it
    explicitly against the strand-collateral failure mode above.

12. An optional protocol fee diverts a configurable fraction of interest accrued
    in each market to a fee recipient account. Both the percentage and the
    recipient are set by the admin authority. The fee fraction is bounded by a
    hard maximum of **25% of accrued interest**, enforced in the immutable core
    (matching Morpho Blue's `MAX_FEE`): the admin may set any value at or below
    this cap, but the core rejects any attempt to exceed it.

13. Each market references the oracle program specified at creation for
    collateral valuation and liquidation triggers. The oracle is a separate LEZ
    program.

14. **Flash loans (LEZ-native).** A user can borrow assets from any market
    within a single transaction. The protocol's flash-loan entrypoint transfers
    the requested loan tokens to the borrower and tail-calls a borrower-supplied
    continuation, handing it an **unforgeable repayment capability** (per
    LP-0015's capability-protected continuation model). The borrower's
    continuation must, before the transaction terminates, tail-call back into
    the lending program's internal `repay_flash` continuation with the
    capability and the principal plus fee. If the transaction does not complete
    the chain with full repayment, the entire transaction reverts atomically and
    no funds leave the market. There is no synchronous reentrancy: control hands
    off fully at each tail call, so the EVM concepts of CEI ordering and
    reentrancy locks do not apply. Flash loans are **zero-fee**, matching Morpho
    Blue's deployed implementation (the deployed `Morpho.sol` collects exactly
    `assets` on repayment with no premium). Repayment must equal the principal
    exactly; no fee is charged.

15. **Reference liquidator.** Ship a reference liquidator daemon that
    continuously monitors all markets and executes liquidations when positions
    exceed their market's LLTV. This is provided as example software for third
    parties to fork and operate; Logos does not commit to running it.
    Liquidation on-chain remains fully permissionless: anyone can write and run
    their own.

16. **Reference risk monitor.** Ship a reference risk-monitoring service that
    tracks protocol health metrics: per-market utilisation, position LTVs
    approaching liquidation thresholds, and oracle feed status. Like the
    liquidator, this is reference software for third parties; it must expose
    metrics via an API or dashboard.

17. **Delegated position management.** An account can authorise another account
    to manage its position on its behalf (supply, withdraw, borrow, repay for
    the authoriser), matching Morpho Blue's `setAuthorization` mechanism.
    Authorisation must be grantable and revocable by the position owner, and the
    program must support a signature-based authorisation path (equivalent to
    `setAuthorizationWithSig`) so a third party can submit the authorising
    transaction. This delegation surface is what the soft-requirement on-chain
    bundler (Soft Functionality 1) composes against; without it the bundler
    cannot act on a user's position in a single atomic chain. Liquidation is not
    gated by authorisation: it remains permissionless per F10.

#### Usability

01. Build the program using the
    [SPEL framework](https://github.com/logos-co/spel), which generates the IDL
    and client code from the program definition.
02. **Logos app architecture.** Deliver the user-facing application as two
    composable modules: (a) a **core module** containing all business logic,
    transaction construction, state querying, and SDK-level operations, with no
    GUI dependencies; and (b) a **GUI module** in QML + C++ that consumes the
    core module and is loadable in the Logos app (Basecamp) via a git repo. The
    core module must be the single source of truth for protocol interaction
    logic and must be directly reusable by headless consumers (CLI, reference
    liquidator, reference risk monitor) without any GUI dependency. The two
    modules must build and ship independently; the GUI must contain no business
    logic that is not also accessible through the core module.
03. Provide local build instructions and downloadable assets for the GUI,
    loadable in the Logos app (Basecamp) via git repo.
04. Provide a CLI that covers core functionality of the program. The CLI
    consumes the same core module as the GUI. The CLI may have fewer features
    than the GUI but must support all essential operations.
05. The reference liquidator (F15) and reference risk monitor (F16) are built on
    the same core module as the GUI and CLI, exposed as headless daemons
    suitable for third parties to fork and run independently. No GUI dependency.
06. The GUI supports creating markets, supplying, borrowing, repaying,
    withdrawing, and viewing position health per market.
07. Position LTV is displayed per market per borrower, queryable on-chain and
    surfaced in both CLI and GUI.
08. Current supply APY, borrow APR, and utilisation are displayed per market in
    the GUI and CLI.
09. When interacting via a private account, the SDK must handle the atomic
    deshield (deposit token + native gas) as a single indivisible user action,
    preventing accidental privacy leaks from externally funding the intermediate
    account.
10. When interacting via a private account, before each operation the GUI must
    show the estimated transaction fee and confirm that the user's shielded
    balance covers both the operation amount and fees within the single deshield
    action. If the balance is insufficient, a clear, actionable error must be
    shown before any transaction is submitted, preventing partial deshields that
    could leave funds stranded in an ephemeral account.
11. The GUI must preview the position LTV impact of a borrow or collateral
    withdrawal before the user confirms: displaying both the current LTV and the
    projected LTV after the operation. (A health check gates borrows and
    collateral withdrawals; withdrawing supplied loan tokens per F5 is gated by
    available market liquidity, not by the borrower's position health, so it
    does not move position LTV.)

#### Reliability

> **Oracle trust model (matching Morpho Blue).** The core is oracle-agnostic: it
> calls the market's oracle and trusts the returned price, with no staleness,
> heartbeat, or confidence check in the core. Oracle quality is the supplier's
> responsibility, evaluated at supply time (per F3's trust model). Staleness and
> manipulation defences, where wanted, live inside the oracle program a market
> creator selects, not in the lending core. The reference risk monitor (F16)
> surfaces oracle-staleness signals off-chain so suppliers and vault curators
> can act; it does not gate the core.

1. If a market's oracle feed becomes permanently unavailable, the program
   rejects only operations that depend on that feed (borrows, liquidations) for
   the affected market. All other markets and operations continue unaffected.
2. Bad debt remaining after a liquidation exhausts available collateral is
   socialised among the market's suppliers (supply shares decrease in value).
   Bad debt in one market does not affect any other market.
3. The following invariants are **formally verified** before mainnet release
   (Certora-style or equivalent), as a precondition of the core program being
   immutable: (a) a position whose LTV is below its market's LLTV cannot be
   liquidated, (b) absent socialised bad debt, supply share value is
   monotonically non-decreasing, (c) the AdaptiveCurveIRM borrow rate is
   monotonically non- decreasing in market utilisation, (d) the protocol cannot
   enter a state where total supplied is less than total borrowed plus on- chain
   liquid balance (solvency).
4. The liquidation incentive is bounded by the protocol's liquidation incentive
   factor formula so that liquidating a position at exactly its market's LLTV is
   profitable for the liquidator yet does not, on its own, push the market into
   bad debt. The bound is enforced in the core program (not configurable per
   market) and is covered by the formal verification of invariant (a).
5. Liquidation of an unhealthy position completes in a single LEZ transaction
   callable by any account. Liquidation liveness does not depend on a privileged
   role, a keeper allowlist, or any off-chain infrastructure: the trigger
   condition is a function of on-chain state and the current oracle price only.
6. Supply, withdraw, repay, and supply-share redemption cannot be paused,
   frozen, or blocked by any admin authority (a consequence of the immutability
   in F2): there is no protocol function that can halt these operations for a
   solvent position.
7. Share accounting MUST resist donation and inflation (first-depositor)
   attacks. The share-to-asset conversion must remain safe when a market's
   liquid balance is manipulated by a direct token transfer, including against
   an empty or near-empty market. Morpho Blue achieves this with virtual shares
   and virtual assets (`VIRTUAL_SHARES`, `VIRTUAL_ASSETS`); applicants may use
   that mechanism or an equivalent, but the resistance property is mandatory and
   must be covered by a test. This is non-negotiable for an immutable core: the
   ecosystem has lost over $130M to share-inflation exploits (see
   [Appendix: Security Record](../appendix/lending-ecosystem.md#security-record-2020-2026)),
   and an immutable program cannot be patched after deployment.

#### Performance

1. Document the transaction size of each operation (market creation, supply,
   borrow, repay, withdraw, liquidate, flash loan) on LEZ. LEZ's block size is
   limited and this budget may change during testnet.
2. The program scales linearly in the number of markets without per-market state
   polluting per-user-operation compute. The actual per-deployment market count
   is bounded by LEZ's per-transaction compute budget (see Platform
   Dependencies, Risks, Compute budget, for the benchmark deliverable). As a
   reference order of magnitude, Morpho Blue runs 200+ active markets on
   Ethereum and Base (2026-04); LEZ's per-tx compute envelope will determine how
   close the singleton can run to that figure. The applicant must report the
   maximum supported market count after benchmarking, not commit to a fixed
   figure at proposal time.

#### Supportability

01. The program is deployed and tested on LEZ testnet.
02. End-to-end integration tests run against a LEZ sequencer (standalone mode)
    and are included in CI.
03. CI must be green on the default branch.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
05. A README documents end-to-end usage: deployment steps, program addresses,
    and step-by-step instructions for interacting with the program via CLI and
    GUI.
06. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the SDK, covering the developer integration journey for supply, borrow,
    repay, withdraw, and liquidation.
07. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the CLI, covering the core operator/user journey.
08. Provide Figma designs or equivalent for the GUI.
09. Provide a privacy and anonymisation properties document covering: what
    on-chain state and transaction data is visible to observers; what data is
    protected when the private account path is used; trust assumptions,
    specifying which guarantees are enforced by the on-chain program and which
    depend on correct client behaviour; and what happens if a user bypasses the
    expected interaction path. See the
    [Privacy Architecture](#privacy-architecture) section below for the baseline
    this document must align with.
10. **Audit programme.** The proposal must include a planned audit programme
    covering the lending program, AdaptiveCurveIRM, bundler (if delivered as a
    soft requirement), reference liquidator, and reference risk monitor. The
    proposal must name at least one tier-1 audit firm the applicant intends to
    engage (for example: OpenZeppelin, Trail of Bits, Spearbit, Cantina,
    ChainSecurity, Certora, Halborn), include the audit budget as a line item in
    the proposal, and include the audit timeline ahead of any mainnet
    recommendation. A second independent audit is strongly preferred for the
    lending program given immutability. Audit reports must be published with the
    codebase before mainnet deployment is recommended.

#### + Privacy

1. The GUI and SDK must support both direct public account interaction and the
   deshield→interact→reshield pattern for private account interaction. When a
   user chooses the private account path, the SDK must enforce the complete
   deshield→interact→reshield pattern; the reshield step must not be skippable.
2. When using the private account path, the GUI must display a pre-confirmation
   summary for each operation that clearly identifies what will be visible
   on-chain (amounts, asset type, market address, ephemeral intermediary
   account) and what will remain private (the originating private account, the
   destination of re-shielded tokens, and any link between separate interactions
   by the same user).
3. When using the private account path, the SDK must validate that the target
   account for re-shielding is a private (shielded) account before submitting
   the transaction, and reject the operation with an explicit error if it is
   not.
4. The ephemeral public account (account A) created during the deshield step
   must never be reused across operations. Each protocol interaction from a
   private account must use a freshly generated account with no prior on-chain
   history. Because LEZ accounts are keypair-derived (Solana-style), generating
   an account with no on-chain history is trivial; preventing reuse, however, is
   an SDK responsibility. Applicants must document how the SDK enforces
   single-use (for example, fresh-keypair-per-operation, a deterministic
   single-use derivation scheme tied to a nonce, or a local registry of consumed
   ephemeral keys) and how it survives client restarts and multi-device usage
   without reuse.

### Soft Requirements

If possible.

#### Functionality

1. On-chain bundler program: a separate LEZ program that composes approve +
   supply + borrow (and similar multi-step user actions) into one atomic
   tail-call chain, modelled on Morpho's
   [`morpho-blue-bundlers`](https://github.com/morpho-org/morpho-blue-bundlers).
   The SDK uses the bundler rather than re-implementing composition off-chain.

### Out of Scope

The following are explicitly excluded from this RFP.

This RFP funds **software**. The mainnet deployment of that software, and the
governance around it, is handled through a separate process. The following are
therefore out of scope:

- Mainnet deployment of the lending program.
- The choice of the LLTV grid that production markets can use.
- The choice of the IRM implementations the admin authority enables on mainnet
  (the oracle for each market is chosen by that market's creator at creation
  time per F3, not gated by the admin authority, so it is not on a mainnet
  approval list).
- Holder, custody, or rotation policy for the admin authority on mainnet.
- Any governance design (token, DAO, multisig) around the admin authority.

Curation layer features, deferred to
[RFP-012](./RFP-012-curated-lending-vaults.md):

- Curated vault abstraction: single-asset deposit vaults allocating across
  multiple markets with a curator-managed strategy
- Vault share tokens (receipt tokens): transferable LEZ fungible tokens
  representing a vault deposit position
- Supply and borrow caps (enforced at the vault layer, not the core)
- Allocator and sentinel roles for vault governance

Other exclusions:

- Alternative IRM implementations beyond AdaptiveCurveIRM. Additional IRMs may
  be enabled later by the admin, but are not deliverables of this RFP.
- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Each market has exactly one collateral token; users compose multi-collateral
  exposure by holding positions across multiple markets, optionally via curated
  vaults (RFP-012).
- Off-chain indexing surface (indexer plus GraphQL API). Morpho ships a
  Messari-schema subgraph stack and a hosted GraphQL API at `api.morpho.org`
  alongside the contracts, which frontends, the Morpho SDK, vault allocators,
  and risk dashboards read from rather than from chain directly. The equivalent
  indexing surface for LEZ is not a deliverable of this RFP and may be covered
  by a future RFP.

### Privacy Architecture

The protocol state (lending markets, positions, interest indices, and token
accounts) lives in **public (on-chain) state**. This is a deliberate
architectural choice: public state enables permissionless liquidation, oracle
integration, and composability without open cryptographic research challenges.

User privacy is optionally enforced at the UX layer. The GUI and SDK must
support both direct public account interaction and private account interaction
via the deshield, interact, reshield pattern: the user acts from a private
account, the SDK deshields to a fresh single-use public account that carries out
the operation, and any outputs are reshielded back to the private account. When
a user opts to interact from a private account, the SDK must enforce the
complete pattern end to end, including atomic funding of gas, so that no
external funding source links the ephemeral account to an existing identity.

The full privacy and anonymisation properties (the complete observable-on-chain
versus protected-data breakdown, trust assumptions, and bypass behaviour) are a
deliverable of this RFP, captured in the privacy and anonymisation properties
document required by Supportability item 9.

## ⚠ Platform Dependencies

This RFP is open for proposal submission. However, development is blocked until
the hard blockers below are resolved. LEZ has similar programming capabilities
to Solana but some primitives required by a lending protocol are not yet
available.

### Hard blockers

These must be available on LEZ before this RFP can open.

#### Oracle provider

No oracle provider is available on LEZ. The lending protocol requires external
price feeds for collateral valuation and liquidation triggers.
[RFP-020](./RFP-020-redstone-oracle-adaptor.md) (RedStone off-chain oracle
adaptor) delivers external price feeds; [RFP-019](./RFP-019-twap-oracle.md)
(on-chain TWAP oracle) is an alternative once a DEX is live.

#### Token authorities (LP-0013)

The lending program is a token custodian: it holds supplied loan tokens per
market, holds collateral on behalf of borrowers, pays liquidators atomically
from collateral, and routes the protocol fee. This requires the
transfer-authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md),
currently **open**.

### Resolved dependencies

These primitives were once blockers but are now delivered on LEZ, so they no
longer gate this RFP. They remain in the frontmatter `dependencies` index for
traceability.

#### General cross-program calls (LP-0015)

General cross-program calls via tail calls let a lending operation transfer
tokens and then continue into a protected continuation to update interest
indices and position state. F14 (flash loans) depends on this mechanism.
[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
is **closed**, delivered by the LEZ team as part of the core runtime.

#### Event emission (LP-0012)

Structured events let the reference liquidator (F15), reference risk monitor
(F16), and any third-party liquidator observe on-chain state changes without
polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
is **closed**.

### Risks

#### Compute budget

LEZ currently processes one private transaction per block (as of 2026-04).
Public LEE documentation that pins this down is still pending; once it lands,
link it from the Resources section. Liquidation is the most compute-intensive
lending operation: it reads the borrower's position, collateral balance, oracle
price, interest indices, and market parameters, then writes updated state. On
Solana, where the per-instruction default is 200,000 compute units and the
per-transaction maximum is 1.4M compute units (see
[Solana compute budget](https://solana.com/docs/core/fees/compute-budget)), a
liquidation typically consumes hundreds of thousands of compute units; the exact
figure for LEZ depends on the implementation. If LEZ's per-transaction compute
budget is insufficient, liquidations will fail and the protocol risks
insolvency.

**Deliverable.** Applicants must benchmark the compute cost of liquidation on
LEZ devnet/testnet once the protocol is integrated and include the figures in
the README. If liquidation does not fit within the per-transaction budget,
applicants must propose a remediation (for example, a two-phase liquidation via
tail-call continuation) and implement it within the funded scope.

## 👤 Recommended Team Profile

Team experienced with:

- Rust program development on LEZ (or SPEL / Solana / SVM, which transfer
  directly)
- DeFi lending protocol design (Morpho Blue, Aave, Compound, Kamino, or similar)
- Interest rate modelling and utilisation curve design (AdaptiveCurveIRM in
  particular)
- Liquidation mechanism design and MEV considerations
- Oracle integration and price feed security
- Formal verification of smart contracts (Certora or equivalent)
- Writing and running on-chain tests against a LEZ sequencer (or equivalent
  local-validator workflows from Solana, e.g. Bankrun, Anchor tests)
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **16 weeks**

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md):
  reference pattern for the bounded admin authority (F11 admin scope, F12
  protocol fee, F14 flash-loan fee). Soft dependency: the bounded admin
  functions should reuse this library once delivered, but the protocol can be
  built without it.
- [Appendix: Lending and Borrowing Ecosystem](../appendix/lending-ecosystem.md):
  ecosystem survey across Ethereum, Solana, BTCFi, and long-tail chains; Morpho
  Blue deep dive

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
