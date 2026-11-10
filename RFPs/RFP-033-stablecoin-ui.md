---
id: RFP-033
title: Reflexive Stablecoin Position Management UI
tier: M
status: draft
category: Applications & Integrations
dependencies:
  - id: RFP-013
    reason: Provides the stablecoin program, core module and CLI, including shared position operations, private account handling and seed recovery; final integration requires versions compatible with the mainnet-equivalent LEZ release targeted by this interface.
---

# RFP-033 — Reflexive Stablecoin Position Management UI

> **Combined proposals.** Proposals covering both this RFP and the separate
> liquidation and auction UI RFP will be given preference. Each scope is
> evaluated separately.

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
> does not make any representation, provide any advice or assume any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

Build the user-facing interface for the reflexive stablecoin of
[RFP-013](./RFP-013-reflexive-stablecoin-protocol.md): a Logos mini-app in which
a user opens a collateralised position, mints the stablecoin against it,
monitors its health, adjusts it, and closes it. The RFP-013 program and its
Logos core module and CLI are specified in RFP-013, including the client-side
private account path, position discovery, recovery and shared calculations. This
RFP covers the mini-app and its integration with those capabilities.

Two things make this more than a form over the program's instructions. First, a
position's health moves even when nothing else does: debt is valued at the
redemption price, which drifts at the redemption rate set by the protocol's
controller, and the stability fee accrues continuously. The interface has to
make that visible and not show a single static number. Second, all position
state on LEZ is public. Privacy comes from the client: funding each operation
from a private account and returning its output to one. A position is a
long-lived account, so this is harder than for a one-shot swap, and the
interface must get it right by default.

A separate RFP covers the liquidation and auction interface over
[RFP-014](./RFP-014-liquidation-auction-engine.md), including manual liquidation
and auction participation alongside automated liquidators. Those screens are
outside this RFP. Proposals should state whether the team intends to apply for
both UI scopes and how they would share components and design.

A team likely to succeed has shipped interfaces for CDP or lending protocols,
has built QML ui modules with C++ core modules, and can reason about what client
behaviour leaks on a public chain.

## 🔥 Why This Matters

A position's collateralisation ratio can change even when its collateral price
does not. In an **illustrative example** in
[Reflexer's FAQ](https://docs.reflexer.finance/faq#how-does-rai-workbehave),
RAI's redemption price rises from $5 to $5.20 while ETH remains at $1,000 after
an earlier price increase. These are hypothetical values used to explain the
mechanism, not observed market data or proposed prices, pegs or parameters for
RFP-013.

With collateral and debt quantities held constant, a higher redemption price
increases the value of the debt relative to the collateral. Stability fees can
also increase the debt over time. The interface therefore needs to distinguish
market price from redemption price, show accrued fees, and explain how these
changes affect the position's distance to liquidation. Previews and projections
let users inspect these effects when opening, adjusting or closing a position.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Open a position: choose the collateral amount and, optionally, an amount of
   stablecoin to mint in the same flow, within the maximum the protocol allows
   for that collateral, and see the resulting collateralisation ratio against
   the protocol minimum before confirming.
2. Manage a position: deposit collateral, withdraw collateral, mint, repay
   partially, repay in full, and close. The withdraw and mint actions show their
   respective maximum amounts while keeping the position at or above the minimum
   collateralisation ratio.
3. Repaying in full and closing must succeed as one user action, even though the
   stability fee keeps accruing between the moment the amount is computed and
   the moment the transaction is included, which on the private path can be
   minutes later (see Performance #1). It must leave no residual debt that
   blocks closing. If this cannot be achieved with the RFP-013 program interface
   as delivered, the proposal must say so and specify the program change it
   needs.
4. List all of the user's positions, on both the public and private paths, with
   collateral, current debt including accrued stability fee, collateralisation
   ratio, and distance to the minimum ratio.
5. Recover all of the user's positions from the user's wallet seed alone on a
   fresh install, with no locally cached state and no separately backed-up
   secret, using the derivation and discovery capabilities of the RFP-013 core
   module. This covers every position created through that module on either
   path, including positions created through its CLI.
6. Show protocol state: market price, redemption price, redemption rate (also as
   an annualised figure), stability fee rate, minimum collateralisation ratio,
   the age of the last price-feed update, and whether the protocol is frozen.
7. When the protocol is frozen or the price feed is stale, actions the program
   rejects in that state are disabled with an explanation, and the rest stay
   available. Where an action is unavailable because a step it depends on is
   blocked, the interface says so.

#### Usability

01. Reuse the RFP-013 C++ core module for position operations, discovery,
    previews, projections, the private account path and recovery. Shared
    functionality belongs in that module and is not reimplemented in the UI. All
    off-chain software described here must be implemented as Logos modules.
02. Provide a Logos mini-app QML GUI (Logos ui module) covering every
    Functionality requirement.
03. Integrate with the same core module and wallet configuration as RFP-013's
    Logos Core headless CLI. Positions created or changed through either
    interface must be discoverable and manageable through the other. A separate
    CLI is not a deliverable of this RFP.
04. Before every operation, show a before/after preview: collateral, debt,
    collateralisation ratio, the redemption price at which the position would
    reach the minimum ratio, and the estimated transaction fee. Compute the
    preview for a stated time and chain-state snapshot using the program's
    documented arithmetic and rounding. Explain that fees and redemption price
    may change before inclusion. After completion, the position view must match
    the resulting chain state.
05. For each position, show how its health changes over time if nothing else
    changes: at the current redemption rate and stability fee, the projected
    collateralisation ratio over a user-selectable horizon, and the time until
    it reaches the minimum ratio, if it does. Explain in the interface that debt
    is valued at the redemption price and that this price moves on its own.
06. Present liquidation risk without hiding the numbers behind colours or
    labels: any status indicator must be accompanied by the ratio, the minimum,
    and the margin between them. Warn the user before confirming an operation
    that leaves the position within a configurable margin of the minimum, with a
    documented default.
07. Support both public and private account paths for every position operation.
    The private path is pre-selected; choosing the public path requires an
    explicit action.
08. Before a user first opens a position on the private path, the interface
    states what remains observable (see Supportability #9), in plain language.
09. Operations that require client-side proving are treated as long-running
    background operations with visible progress. The interface stays usable
    while they run, and the user can see the state of each operation in flight.
10. Failed operations return clear, actionable error messages that name the
    constraint that was violated (minimum ratio, frozen protocol, stale price
    feed, insufficient balance, and so on).
11. The user can change the LEZ sequencer or node the interface connects to.
12. Signing goes through the LEZ core module wallet (see Platform Dependencies,
    "Wallet"). The interface holds no key material of its own beyond what the
    wallet exposes for the private account path.

#### Reliability

1. An operation on the private path that is interrupted at any step (closed
   application, lost connection, rejected transaction) never strands funds in an
   intermediate public account. On restart, the interface detects the incomplete
   operation through the RFP-013 core module and either completes it or returns
   the funds to the user's private account, and shows the user which it did.
2. Displayed position and protocol values are derived from on-chain state using
   the same fixed-point arithmetic as the program, and are refreshed after every
   operation and at a regular interval stated in the proposal. The proposal must
   justify the interval and describe how stale or unavailable data is indicated
   to the user.
3. No mandatory external services: beyond the user's configured LEZ sequencer
   and node, the interface depends on no third-party service. Any optional
   service is disclosed, can be disabled, and the interface stays fully
   functional without it. The interface sends no user account, position or
   balance data to any party other than the configured sequencer and node,
   including through optional services. Analytics require explicit opt-in and
   must exclude those data.
4. A rejected or failed transaction leaves the interface's view of the position
   consistent with chain state.

#### Performance

1. Proposals must state end-to-end latency targets for each position operation
   on both paths, identifying hardware and network assumptions. At delivery,
   measure and document, with real proving, the end-to-end time of each position
   operation on the public and the private paths, from confirmation to the
   position view reflecting the result. Figures produced in development mode
   will not be accepted.
2. Listing the user's positions and recovering them from seed (Functionality #4,
   #5) complete in a time the proposal states and the delivery measures, for a
   user with at least 20 positions.

#### Supportability

01. Deliver and test the interface with compatible RFP-013 program and core
    module versions on a mainnet-equivalent LEZ release, accounting for the
    breaking changes expected in 0.5.0. The proposal must identify the target
    release and compatibility assumptions; the final test report must record the
    exact versions used. Testing only against 0.4 is insufficient. This
    requirement does not require mainnet deployment.
02. End-to-end tests of every Functionality requirement, on both paths, run
    against a LEZ sequencer (standalone mode) with the RFP-013 program deployed,
    and are included in CI.
03. Automated UI tests cover the open, mint, repay in full and close, and
    recovery flows, and run in CI.
04. CI must be green on the default branch.
05. Every hard requirement in Functionality, Usability, Reliability and Privacy
    has at least one corresponding test.
06. A README documents installing the mini-app in Basecamp, connecting to the
    target network, and each user flow, with links to RFP-013 CLI instructions
    and examples of using the same positions across the CLI and mini-app.
07. Document the mini-app integration with the RFP-013 core module in repo,
    referencing its API documentation and recording required versions.
08. Provide Figma designs or equivalent for all mini-app screens and states,
    including warning, frozen, stale-feed and in-progress states. Include a
    prototype review before implementation and a usability-test-and-revision
    checkpoint before final acceptance. Proposals must state the participant
    profile, number of participants and iteration rounds, tasks, and measurable
    acceptance criteria. Test opening a position, interpreting liquidation risk,
    repaying and closing, and recovering from seed. Deliver findings, revisions
    and a retest of failed tasks; record unresolved issues and their impact.
09. Provide a privacy properties document covering what each operation reveals
    on-chain on each path, what the configured node learns from the interface's
    queries, which guarantees depend on correct client behaviour, and what
    happens if a user bypasses the interface. Cover linkage of operations on one
    position, amount and timing correlation between private funding, the
    operation and the return to a private account, and inference from queried
    account sets. For each residual correlation, state why it remains and how
    the design reduces it.
10. Publish the resulting modules in a module catalog of the team's own, built
    from the
    [Logos module catalog template](https://github.com/logos-co/logos-modules-release-base),
    so the modules are installable by Logos clients.
11. The modules are built and released to that catalog using
    [logos-modules-release-action](https://github.com/logos-co/logos-modules-release-action),
    and install and run on Linux and macOS.
12. Provide a design decision log in repo recording each major design decision
    and choice: the context, the options considered, the option chosen, and the
    rationale and trade-offs. The log is updated as decisions are made and is
    current at each milestone delivery.

#### + Privacy

1. On the private path, collateral and fees are funded from the user's private
   account, and minted stablecoin and withdrawn collateral end in a private
   account of the user. The interface must not offer a completion path that
   leaves these outputs public.
2. On the private path, a public owner account is used for that position only.
   Accounts used to fund or pay fees for that position must not be shared with
   other positions or the user's public account activity.
3. On the private path, no ownership, funding or fee-payment relationship links
   those accounts on-chain to the user's private account, other positions or
   public accounts. Residual inference from public amounts, timing and node
   queries must be documented under Supportability #9; this requirement does not
   claim to eliminate those correlations.

### Soft Requirements

1. History per position (collateral, debt and ratio over time) and protocol
   history (market price, redemption price, redemption rate), built from LEZ
   node data without a mandatory third-party service.
2. Options that reduce timing correlation on the private path, such as delaying
   the return to a private account, with the trade-off shown to the user.
3. Once the RFP-014 engine is integrated with RFP-013, show a position that has
   been liquidated: what was seized, the debt cleared, and anything returned.
4. Shared components and design tokens structured so the companion liquidation
   and auction interface can reuse them, consistent with the
   [Logos design system](https://github.com/logos-co/logos-design-system).
5. Notify the user, while the mini-app is running, when a position crosses the
   warning margin of Usability #6.

### Out of Scope

- Changes to the on-chain RFP-013 program. Proposals must identify any program
  change needed to meet a UI requirement, including full repayment and closure,
  and explain the resulting dependency and delivery impact.
- Reimplementation of the core module, CLI, private account path, discovery,
  recovery or shared calculations specified in RFP-013.
- Liquidation and auction screens, including manual liquidation and bidding;
  these belong to the separate UI RFP over RFP-014.
- Keeper bots and automated liquidation services.
- Notifications while the mini-app is closed. In-app warnings remain in scope;
  notifications while it is running are Soft Requirement #5.
- Mainnet deployment or operation of a hosted service.

## ⚠ Platform Dependencies

### Hard dependencies

#### Reflexive Stablecoin Protocol (RFP-013)

This interface operates the RFP-013 program and calls it through the RFP-013
Logos core module. The program is implemented in
[logos-blockchain/lez-programs](https://github.com/logos-blockchain/lez-programs)
(`programs/stablecoin`, design in `docs/stablecoin/README.md`). RFP-013 owns the
core module and CLI, including shared calculations, private account operations
and seed recovery. The program's interface relevant to this RFP:

- Positions are public accounts addressed by `(owner, nonce)`, each with its own
  collateral vault. Every later operation on a position must be signed by its
  owner. There is no on-chain index of positions, so clients find them by
  deriving addresses from known owners and nonces. A nonce cannot be reused
  after a position is closed.
- Collateralisation is checked against the redemption price, not the market
  price. The price feed is used by the controller and as a staleness gate on
  minting.
- Repayment burns an exact amount and rejects overpayment.

This RFP does not include changes to the RFP-013 program. Where a requirement
here cannot be met with the program as delivered, the proposal must identify it.

### Soft dependencies

#### Wallet

Signing, account selection and proving go through the Logos wallet. Today that
is the LEZ core module
([logos-execution-zone-module](https://github.com/logos-blockchain/logos-execution-zone-module)),
which Basecamp shares across mini-apps.
[LP-0021](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0021.md)
specifies a Wallet Provider SDK with account selection and approval prompts. The
interface should use it once available, and proposals should state how they move
to it.

#### Price feed

The interface reads whichever price feed the RFP-013 deployment is configured
with, either [RFP-019](./RFP-019-twap-oracle.md) (on-chain TWAP) or
[RFP-020](./RFP-020-redstone-oracle-adaptor.md) (RedStone adaptor). It does not
integrate a feed of its own.

#### History

If history is included, it must be built from LEZ node data without a mandatory
third-party service.

### Risks

#### Private account integration

Position-owner accounts persist across operations. The interface must use the
RFP-013 core module's derivation, discovery and private account operations
consistently, including after interruption or a fresh installation. Proposals
must identify any missing upstream capability and its delivery impact.

#### Upstream compatibility and schedule

Final end-to-end acceptance depends on compatible versions of the RFP-013
program and core module on the target LEZ release. Proposals must identify
interface assumptions and how breaking changes would affect scope and schedule.
Designs and integration may begin against a local sequencer, but do not replace
validation on the target release.

## 👤 Recommended Team Profile

Team experienced with:

- Interfaces for CDP, lending or other leveraged DeFi protocols
- QML ui module and C++ core module integration
- Product design for financial risk (presenting liquidation risk, previews,
  confirmation flows)
- Wallet integration and client-side transaction construction
- Privacy analysis of client behaviour on public chains

## ⏱ Timeline Expectations

Estimated duration: **8–12 weeks**, excluding time spent waiting for the RFP-013
program and core module to be available on the target LEZ release. Proposals
must include the design review, usability testing and revision checkpoints
within their schedule.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-013: Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
- [RFP-014: Liquidation & Auction Engine](./RFP-014-liquidation-auction-engine.md)
- [RFP-013 program design](https://github.com/logos-blockchain/lez-programs/blob/main/docs/stablecoin/README.md)
- [logos-blockchain/lez-programs](https://github.com/logos-blockchain/lez-programs):
  the RFP-013 program and core module, and the DEX mini-app (`apps/amm`), a
  reference for a QML mini-app over a LEZ program
- [Logos Module Builder](https://github.com/logos-co/logos-module-builder):
  builds core and QML ui modules
- [Logos Design System](https://github.com/logos-co/logos-design-system)
- [LP-0021: LEZ Wallet and Provider SDK](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0021.md)
- [Introduction to the Logos Execution Zone](https://docs.logos.co/lez): the
  public/private account model
- [RAI Whitepaper](https://github.com/reflexer-labs/whitepapers/blob/master/English/rai-english.pdf)

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
