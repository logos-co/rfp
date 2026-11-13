---
note: This metadata is for internal programme tracking only. It is not part of this specification and does not form part of its requirements.
id: RFP-008
title: Lending & Borrowing Protocol
tier: XL
status: closed
category: Applications & Integrations
dependencies:
  - id: LP-0013
    reason: Token transfer authorities are required to hold supplied assets and collateral and pay liquidators atomically; LP-0013 describes this capability.
  - id: RFP-020
    reason: Price feeds are required for collateral valuation and liquidation triggers (F13); RFP-020 and RFP-019 are illustrative examples.
  - id: LP-0015
    reason: General cross-program calls, delivered on testnet 0.4 as synchronous messages with replies; used to compose token transfers with state updates, by the flash loan model (F14) and by the state reads other programs rely on (F18).
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
> and (iii) complying with any legal and regulatory requirements that apply to
> them. Software developed under the Program is published and maintained by its
> developers, not by Logos.
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
> does not make any representation, provide any advice or assume any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

This specification describes a permissionless, isolated-market lending protocol
on LEZ, following the Morpho Blue design: each market is an independent lending
pair defined by five immutable parameters (loan token, collateral token, oracle,
interest rate model, liquidation LTV). Anyone can create a market from enabled
parameters; the core protocol is minimal, non-upgradeable, and governance-free
at the market level. The protocol includes the AdaptiveCurveIRM interest rate
model, permissionless liquidation, oracle-fed price data, and flash loans built
on LEZ's synchronous cross-program messages.

This RFP supports development of the **software** of a Morpho Blue equivalent on
LEZ: market creation, supply, borrow, repay, withdraw, liquidation, interest
accrual, oracle integration and flash loans, plus reference implementations of a
liquidator and risk monitor for third parties to fork and operate independently.
The program targets LEZ testnet 0.4. Mainnet deployment, production parameters,
admin role holders and governance are outside this RFP's scope and are matters
for independent deployers or operators. The curation layer (single-asset deposit
vaults that allocate across multiple markets, vault share tokens, supply and
borrow caps) is specified separately in
[RFP-012](./RFP-012-curated-lending-vaults.md), which also describes a Lending
Market Interface through which curated vaults can allocate to a conforming
lending protocol (F19).

## 🔥 Why This Matters

An isolated-market lending program lets accounts supply a loan token to a
specific market and borrow it against that market's collateral token. Each
market has its own accounting and immutable parameters. Bad debt is allocated to
suppliers of the affected market rather than shared across the program's
markets; this accounting separation does not eliminate risks from shared code,
assets or oracle dependencies.

Market creation does not require admin approval once the IRM and LLTV are
enabled. The market creator selects the oracle, and suppliers assess the
market's parameters before supplying. The core exposes operations that other
programs, including curated vaults, can compose with. Its non-upgradeable design
makes testing, formal verification and published audit reports relevant
development deliverables.

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
    token, oracle, IRM, LLTV) cannot be changed. On testnet 0.4, the program is
    deployed with the program loader's `immutable` flag set, after which its
    bytecode cannot be updated.

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
    The IRM program answers rate queries (supply APY and borrow APR for a given
    market id) by replying to the caller, with no state mutation. To avoid a
    message round trip on every accrual, applicants may also link the same IRM
    code into the lending program, provided a test shows that both return
    identical rates.

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
    The liquidation incentive is `(LIF − 1) × repaid debt`, before transaction
    costs and execution effects, paid out of the borrower's collateral. The
    liquidation incentive factor (LIF) is derived from the market's LLTV via the
    Morpho Blue formula: `LIF = min(M, 1 / (β·LLTV + (1−β)))` with cursor β =
    0.3 and maximum bound M = 1.15. Higher LLTV markets yield smaller
    incentives; lower LLTV markets yield larger incentives, capped at M.
    Reference:
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
    values to support iteration. The identity of the admin authority holder,
    production LLTV and IRM selections, fee settings and recipient, and related
    governance are matters for the independent deployer or operator and are
    outside this RFP's scope.

    **No emergency pause is deliberate.** Following the Morpho Blue model, the
    core exposes no admin function to freeze funds or halt liquidations. This
    does not guarantee that operations remain executable: oracle failures, code
    defects or asset behaviour can still prevent them. Pausing a market with
    active borrows would prevent liquidations and could strand collateral.
    Market accounting is isolated (F3); the separately specified curated vault
    layer (RFP-012) provides a sentinel role that can revoke a market's cap and
    stop new allocations without changing the core. The reference risk monitor
    (F16) reports oracle-staleness and unhealthy-market signals. These
    mechanisms do not guarantee loss prevention. Applicants proposing a
    different trade-off (e.g. a narrow oracle-halt flag scoped to a single
    market) must justify it explicitly against the strand-collateral failure
    mode.

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
    the requested loan tokens to the borrower and sends a message to a
    borrower-supplied callback. Once the callback, and every message it sends,
    has been processed, the lending program verifies within the same transaction
    that the principal has been returned to the market. If it has not, the
    operation fails and the entire transaction reverts, including the transfer
    to the borrower. On testnet 0.4, messages are processed depth first in the
    order they are sent, so a check sent after the callback runs after it (see
    the LEZ flash swap example in Resources).

    The callback may call back into the lending program: the runtime has no
    re-entrancy guard, and the lending program's state is written before the
    messages it sends are processed. Applicants must document which operations
    may run during a flash-loan callback and why accounting stays consistent
    when they do. A check on the market's token balance alone is not sufficient
    if the callback can credit the same holding through another operation (for
    example, a supply to the same market); the design must rule this out or
    account for it.

    Flash loans are **zero-fee**, matching Morpho Blue's deployed implementation
    (the deployed `Morpho.sol` collects exactly `assets` on repayment with no
    premium). Repayment must equal the principal exactly; no fee is charged.

15. **Reference liquidator.** Ship a reference liquidator daemon that
    continuously monitors all markets and executes liquidations when positions
    exceed their market's LLTV. This is provided as example software for third
    parties to fork and operate independently; Logos does not operate it.
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
    cannot act on a user's position in a single atomic transaction. Liquidation
    is not gated by authorisation: it remains permissionless per F10.

18. **State reads and replies for other programs.** Other LEZ programs can read
    market and position state within a transaction. On testnet 0.4 a program
    cannot return a value to its caller directly: it sends the value back in a
    message that is processed within the same transaction. The lending program
    answers read messages for market state and for a position's supply shares,
    borrow shares, collateral and their values in token units (including
    interest accrued up to the current block). Each operation that changes a
    position (supply, withdraw, borrow, repay, collateral deposit and
    withdrawal, liquidation) also replies to its caller with the assets and
    shares it moved. A caller can check that a reply comes from the lending
    program and answers the request it sent.

19. **Lending Market Interface.** The program implements the Lending Market
    Interface, an open specification of how curated vaults allocate to and read
    from a lending market, whose intended scope is described in
    [RFP-012](./RFP-012-curated-lending-vaults.md) (Functionality #44 to #50).
    Where the specification publishes a conformance test suite, the program's CI
    runs it. If the specification is not published when work starts, the
    proposal identifies how adoption would be scheduled (for example, as a
    separate milestone) and the effect on scope, phasing and pricing. Applicants
    may comment on the specification while it is open for public comment.

#### Usability

01. Build the program using the
    [SPEL framework](https://github.com/logos-co/spel), which generates the IDL
    and client code from the program definition.
02. **Logos app architecture.** Deliver the user-facing application as two
    composable modules: (a) a **core module** containing all business logic,
    transaction construction, state querying, and SDK-level operations, with no
    GUI dependencies; and (b) a **GUI module** in QML + C++ that consumes the
    core module and is loadable in the Logos app (Basecamp) via the team's own
    git repo. The core module must be the single source of truth for protocol
    interaction logic and must be directly reusable by headless consumers (CLI,
    reference liquidator, reference risk monitor) without any GUI dependency.
    The two modules must build and ship independently; the GUI must contain no
    business logic that is not also accessible through the core module.
03. Provide local build instructions and downloadable assets for the GUI,
    loadable in the Logos app (Basecamp) via the team's own git repo.
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
    so the operation is funded entirely from the user's shielded balance.
10. When interacting via a private account, before each operation the GUI must
    show the estimated transaction fee and confirm that the user's shielded
    balance covers both the operation amount and fees within the single deshield
    action. If the balance is insufficient, a clear, actionable error must be
    shown before any transaction is submitted.
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
   This loss allocation does not debit the supply shares of other markets.
3. The implementing team must have the following invariants **formally
   verified** (Certora-style or equivalent) and publish the verification reports
   alongside the codebase before any production deployment of the software: (a)
   a position whose LTV is below its market's LLTV cannot be liquidated, (b)
   absent socialised bad debt, supply share value is monotonically
   non-decreasing, (c) the AdaptiveCurveIRM borrow rate is monotonically non-
   decreasing in market utilisation, (d) the protocol cannot enter a state where
   total supplied is less than total borrowed plus on- chain liquid balance
   (solvency).
4. The liquidation incentive is bounded by the protocol's liquidation incentive
   factor formula. At the market's LLTV boundary, the calculation must leave
   sufficient collateral value at the oracle price to cover the remaining debt
   after the incentive is paid; this is an accounting property, not a guarantee
   of execution profitability or solvency under changing prices. The bound is
   enforced in the core program (not configurable per market) and is covered by
   the formal verification of invariant (a).
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

1. Document the transaction size and execution gas of each operation (market
   creation, supply, borrow, repay, withdraw, liquidate, flash loan) on LEZ.
   LEZ's block size and gas limits are bounded and may change during testnet.
2. The program scales linearly in the number of markets without per-market state
   polluting per-user-operation compute: the gas of an operation on one market
   does not grow with the number of markets or positions. Any remaining bound on
   the per-deployment market count comes from LEZ's per-transaction execution
   gas limit (see Platform Dependencies, Risks, Execution gas, for the benchmark
   deliverable). As a reference order of magnitude, Morpho Blue runs 200+ active
   markets on Ethereum and Base (2026-04). The applicant must report the maximum
   supported market count after benchmarking, not commit to a fixed figure at
   proposal time.

#### Supportability

01. The team deploys and tests the program on LEZ testnet 0.4.
02. End-to-end integration tests run against a LEZ sequencer (standalone mode)
    and are included in CI.
03. CI must be green on the default branch.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
05. A README documents end-to-end usage: deployment steps, program addresses,
    and step-by-step instructions for interacting with the program via CLI and
    GUI.
06. Publish SDK documentation in the team's own repository, covering the
    developer integration journey for supply, borrow, repay, withdraw and
    liquidation.
07. Publish CLI documentation in the team's own repository, covering the core
    operator/user journey.
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
    the proposal, and include the audit timeline. The implementing team is
    responsible for arranging the audits. A second independent audit is strongly
    preferred for the lending program given immutability. Audit reports must be
    published with the codebase before any production deployment of the
    software.

#### + Privacy

1. The GUI and SDK must support both direct public account interaction and the
   deshield→interact→reshield pattern for private account interaction. When a
   user chooses the private account path, the SDK must complete the full
   deshield→interact→reshield pattern for the operation.
2. When using the private account path, the GUI must display a pre-confirmation
   summary for each operation that clearly identifies what will be visible
   on-chain (amounts, asset type, market address, ephemeral intermediary
   account) and what is not recorded in public on-chain state (the originating
   private account and the destination of re-shielded tokens). The summary must
   explain that public timing and amounts may still allow observers to correlate
   interactions.
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
   transaction, modelled on Morpho's
   [`morpho-blue-bundlers`](https://github.com/morpho-org/morpho-blue-bundlers).
   The SDK uses the bundler rather than re-implementing composition off-chain.

### Out of Scope

This RFP supports development of software. The following are out of scope:

- Mainnet deployment. Mainnet deployment is not part of this RFP: whether, when
  and by whom the software is deployed is a matter for those who deploy it.
- The choice of the LLTV grid that production markets can use.
- The choice of the IRM implementations the admin authority enables on mainnet
  (the oracle for each market is chosen by that market's creator at creation
  time per F3, not gated by the admin authority, so it is not on a mainnet
  approval list).
- Holder, custody, or rotation policy for the admin authority on mainnet.
- Any governance design (token, DAO, multisig) around the admin authority.
- The identity of persons holding the admin authority, the exercise of its
  powers and the selection of production fee settings and recipients are matters
  for the relevant independent deployer or operator.

Curation layer features, specified separately in
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
  indexing surface for LEZ is not a deliverable of this RFP.

### Privacy Architecture

The protocol state (lending markets, positions, interest indices, and token
accounts) lives in **public (on-chain) state**. This is a deliberate
architectural choice: public state enables permissionless liquidation, oracle
integration and cross-program composition.

The private account interaction path is implemented by the GUI and SDK. The GUI
and SDK must support both direct public account interaction and private account
interaction via the deshield, interact, reshield pattern: the user acts from a
private account, the SDK deshields to a fresh single-use public account that
carries out the operation, and any outputs are reshielded back to the private
account. When a user opts to interact from a private account, the SDK must
complete the full deshield-interact-reshield pattern for the operation,
including atomic funding of gas, so that the operation is funded entirely from
the user's shielded balance and the ephemeral account carries no prior history.
Timing and amounts remain public and may be correlated; the pattern does not
guarantee that separate interactions cannot be linked.

The full privacy and anonymisation properties (the complete observable-on-chain
versus protected-data breakdown, trust assumptions, and bypass behaviour) are a
deliverable of this RFP, captured in the privacy and anonymisation properties
document required by Supportability item 9.

## ⚠ Platform Dependencies

Implementation depends on the following technical capabilities on LEZ testnet
0.4. References to RFPs and LPs describe capabilities or provide illustrative
examples; they do not require a particular third-party implementation. Proposals
must identify the implementation and interfaces they build against, any
availability assumptions, and how interface changes would be absorbed in scope,
phasing and pricing. LEZ runtime compatibility remains required.

1. **Price feeds** for collateral valuation and liquidation triggers (F13).
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md) describes an off-chain oracle
   adaptor; [RFP-019](./RFP-019-twap-oracle.md) describes a TWAP oracle that
   depends on DEX price observations.
2. **Token transfer authorities** so the program can hold supplied loan tokens
   and collateral, pay liquidators atomically and route protocol fees according
   to its programmed rules.
   [LP-0013](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
   describes this capability.
3. **General cross-program calls** to compose transfers and state updates and
   implement F14 and F18. On testnet 0.4, a program calls another by sending it
   a synchronous message, processed within the same transaction. Messages are
   processed depth first, in the order they are sent. The called program returns
   a value by sending a message back to its caller, and any failure reverts the
   whole transaction. The runtime puts no fixed limit on the number of messages
   in a transaction; they are bounded by the transaction's execution gas (see
   Risks). A program's state is written before the messages it sends are
   processed, and the runtime has no re-entrancy guard, so a called program can
   call back into its caller. This is a LEZ runtime capability described in
   [LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md).
4. **Structured event emission** so reference and third-party liquidators and
   risk monitors can observe on-chain changes without polling every account. On
   testnet 0.4, a program emits events as part of its response to each message.
   This capability is described in
   [LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md).
5. **A token program and development framework compatible with testnet 0.4.** At
   the time of writing, the LEZ token program
   ([`lez-programs`](https://github.com/logos-blockchain/lez-programs)) and the
   [SPEL framework](https://github.com/logos-co/spel) (Usability 1) target LEZ
   v0.2, and a SPEL port to testnet 0.4 is in progress. Proposals should state
   how they would proceed if either is not available for testnet 0.4 when work
   starts.

### Risks

#### Execution gas

On testnet 0.4, every program a public transaction runs draws on one execution
gas budget (zkVM cycles), set by the transaction's gas limit. That limit cannot
exceed the protocol maximum (`MAX_GAS_EXEC`, 10,000,000 cycles at LEZ
`v0.4.0-rc3`), which is also the cap on total execution gas in a block: a
transaction that uses the full allowance occupies a whole block. Liquidation is
the most compute-intensive lending operation: it reads the borrower's position,
collateral balance, oracle price, interest indices, and market parameters, then
writes updated state. If a liquidation does not fit within the gas limit, or
needs so much of it that it competes for whole blocks, liquidations will fail or
be delayed and the protocol risks insolvency.

Private execution is not metered by gas, but it is constrained by block space.
At `v0.4.0-rc3`, a block's storage budget (`MAX_GAS_STOR`, 1,000,000 bytes) and
the fixed proof charge of about 223 KB per private transaction allow at most
four private transactions per block, each carrying at most eight private
actions. This bounds the throughput of the private account path (Privacy 1 to
4). These constants may change during testnet.

**Deliverable.** Applicants must benchmark the execution gas of liquidation on
LEZ testnet 0.4 once the protocol is integrated and include the figures in the
README. If liquidation does not fit within the per-transaction limit with
headroom, applicants must propose a remediation (for example, a two-phase
liquidation across transactions), explain how it preserves Reliability 5, and
implement it within the funded scope.

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

A track record of operating a lending protocol that holds mainnet TVL on another
chain is a strong advantage.

## ⏱ Timeline Expectations

Estimated duration: **16 weeks**

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md):
  reference pattern for the bounded admin authority (F11 admin scope, F12
  protocol fee). This is an illustrative reference; the protocol does not
  require a particular implementation of this library.
- [Appendix: Lending and Borrowing Ecosystem](../appendix/lending-ecosystem.md):
  ecosystem survey across Ethereum, Solana, BTCFi, and long-tail chains; Morpho
  Blue deep dive
- [LEZ flash swap example](https://github.com/logos-blockchain/logos-execution-zone/blob/v0.4.0-rc3/lee/state_machine/test_methods/guest/src/bin/flash_swap_initiator.rs):
  the testnet 0.4 pattern for a transfer out, a callback and a balance check
  after it, within one transaction (F14)

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
