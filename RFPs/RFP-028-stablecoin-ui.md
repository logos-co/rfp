---
id: RFP-028
title: Reflexive Stablecoin Position Management UI
tier: M
status: draft
category: Applications & Integrations
dependencies:
  - id: RFP-013
    reason: Provides the stablecoin LEZ program and its Logos core module that this interface operates; the UI has nothing to drive until RFP-013 is deployed on LEZ testnet 0.4.
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-028 — Reflexive Stablecoin Position Management UI

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
Logos core module are being built separately and do not include this interface,
the CLI, or the client-side private account path. This RFP covers those.

Two things make this more than a form over the program's instructions. First, a
position's health moves even when nothing else does: debt is valued at the
redemption price, which drifts at the redemption rate set by the protocol's
controller, and the stability fee accrues continuously. The interface has to
make that visible and not show a single static number. Second, all position
state on LEZ is public. Privacy comes from the client: funding each operation
from a private account and returning its output to one. A position is a
long-lived account, so this is harder than for a one-shot swap, and the
interface must get it right by default.

A separate RFP will cover the liquidation and auction interface over
[RFP-014](./RFP-014-liquidation-auction-engine.md). The two are kept apart so
each can be evaluated on its own, but Logos would prefer one team to deliver
both, so that the vault and auction surfaces share one design and one set of
components. Proposals should state whether the team intends to apply for both.

The LEZ program this interface operates is delivered for LEZ testnet 0.4, and
this interface is delivered and tested against that deployment (Supportability
#1).

A team likely to succeed has shipped interfaces for CDP or lending protocols,
has built desktop applications in Qt/QML or a comparable native toolkit, and can
reason about what client behaviour leaks on a public chain.

## 🔥 Why This Matters

A stablecoin that only a CLI user can mint has no users. The RFP-013 protocol
gives LEZ a stable unit of account that does not depend on an issuer, but people
reach it through an interface. In a CDP, a user who misreads how close their
position is to liquidation, or how fast its debt grows, loses collateral.

RAI-style stablecoins are harder to read than a fixed-peg CDP. The target price
is not 1.00 and changes over time, and the liquidation threshold moves with it.
An interface that presents these mechanics plainly is a precondition for the
stablecoin being used at all, and for it then becoming collateral and liquidity
for the DEX ([RFP-004](./RFP-004-privacy-preserving-dex.md)) and lending
([RFP-008](./RFP-008-lending-borrowing-protocol.md)) protocols.

The protocol keeps positions public so liquidation stays permissionless. Whether
a user's position can be traced back to their private balance therefore depends
on how the client funds, signs, and returns each operation.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Open a position: choose the collateral amount and, optionally, an amount of
   stablecoin to mint in the same flow, and see the resulting collateralisation
   ratio against the protocol minimum before confirming.
2. Manage a position: deposit collateral, withdraw collateral, mint, repay
   partially, repay in full, and close. Each action shows the maximum amount
   that can be withdrawn or minted while keeping the position at or above the
   minimum collateralisation ratio.
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
   secret. Losing the key that controls a position means losing its collateral,
   so this applies to every account the interface creates to own a position.
6. Show protocol state: market price, redemption price, redemption rate (also as
   an annualised figure), stability fee rate, minimum collateralisation ratio,
   the age of the last price-feed update, and whether the protocol is frozen.
7. When the protocol is frozen, actions that the program blocks while frozen
   (opening, minting, withdrawing) are disabled with an explanation, and actions
   that remain allowed (depositing, repaying, closing) stay available. When the
   price feed is stale, minting is disabled with an explanation.
8. Support both a public account path and a private account path for every
   position operation, per the Privacy requirements below.

#### Usability

01. Build all client functionality (position discovery, previews and
    projections, the private account path, recovery) in a Logos core module,
    reusing the RFP-013 core module for program calls, so that other Logos ui
    modules and the CLI can use it. All off-chain software described here must
    be implemented as Logos modules.
02. Provide a Logos mini-app QML GUI (Logos ui module) covering every
    Functionality requirement.
03. Provide a CLI that uses the Logos core headless framework and covers the
    position operations in Functionality #1 to #5. The CLI may have fewer views
    than the mini-app.
04. Before every operation, show a before/after preview: collateral, debt,
    collateralisation ratio, the redemption price at which the position would
    reach the minimum ratio, and the estimated transaction fee. Values shown in
    the preview must match the on-chain result up to the program's documented
    rounding.
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
07. The private account path is pre-selected for every operation. Choosing the
    public path requires an explicit action.
08. Before a user first opens a position on the private path, the interface
    states what remains observable (see Privacy #4), in plain language.
09. Operations that require client-side proving are treated as long-running
    background operations with visible progress. The interface stays usable
    while they run, and the user can see the state of each operation in flight.
10. Failed operations return clear, actionable error messages that name the
    constraint that was violated (minimum ratio, frozen protocol, stale price
    feed, insufficient balance, and so on).
11. The user can change the LEZ sequencer or node the interface connects to.
12. Signing and account selection go through the Logos wallet and its approval
    flow (see Platform Dependencies, "Wallet"). The interface holds no key
    material of its own beyond what the wallet exposes for the private account
    path.

#### Reliability

1. An operation on the private path that is interrupted at any step (closed
   application, lost connection, rejected transaction) never strands funds in an
   intermediate public account. On restart, the interface detects the incomplete
   operation and either completes it or returns the funds to the user's private
   account, and shows the user which it did.
2. Displayed position and protocol values are derived from on-chain state using
   the same fixed-point arithmetic as the program, and are refreshed after every
   operation and at a regular interval.
3. No mandatory external services: beyond the user's configured LEZ sequencer
   and node, the interface depends on no third-party service. Any optional
   service is disclosed, can be disabled, and the interface stays fully
   functional without it. No analytics unless strictly opt-in.
4. A rejected or failed transaction leaves the interface's view of the position
   consistent with chain state.

#### Performance

1. Measure and document, with real proving, the end-to-end time of each position
   operation on the public and the private paths, from confirmation to the
   position view reflecting the result. Figures produced in development mode
   will not be accepted.
2. Listing the user's positions and recovering them from seed (Functionality #4,
   #5) complete in a time the proposal states and the delivery measures, for a
   user with at least 20 positions.

#### Supportability

01. Delivered and tested against the RFP-013 program on LEZ testnet 0.4.
02. End-to-end tests of every Functionality requirement, on both paths, run
    against a LEZ sequencer (standalone mode) with the RFP-013 program deployed,
    and are included in CI.
03. Automated UI tests cover the open, mint, repay in full and close, and
    recovery flows, and run in CI.
04. CI must be green on the default branch.
05. Every hard requirement in Functionality, Usability, Reliability and Privacy
    has at least one corresponding test.
06. A README documents installing the mini-app in Basecamp, connecting to
    testnet, and each user flow, with step-by-step instructions for the CLI.
07. Provide documentation in repo for the core module, covering the developer
    integration journey.
08. Provide Figma designs or equivalent for all mini-app screens and states,
    including warning, frozen, stale-feed and in-progress states.
09. Provide a privacy properties document covering what each operation reveals
    on-chain on each path, what the configured node learns from the interface's
    queries, which guarantees depend on correct client behaviour, and what
    happens if a user bypasses the interface.
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

1. On the private path, collateral and gas are funded from the user's private
   account and every output (minted stablecoin, withdrawn collateral) is
   returned to a private account of the user. The return step cannot be skipped,
   and the interface rejects a return target that is not a private account.
2. The public account that owns a position on the private path is used for that
   position only. No on-chain data links it to the user's private account, to
   the owner of any other position of the same user, or to the user's public
   accounts.
3. Each deshield transfers the operation token and the gas for the operation in
   one action, so the account is never funded from a second, linkable source.
4. Proposals must document every correlation the design leaves observable,
   including at least: that all operations on one position are linkable to each
   other through the position; amount and timing correlation between a deshield
   and the operation it funds, and between an operation and its return to a
   private account; and what the configured node can infer from the set of
   accounts the interface queries. For each, state why it remains and what the
   design does to reduce it.
5. The interface sends no user account, position or balance data to any party
   other than the user's configured LEZ sequencer and node.

### Soft Requirements

1. History per position (collateral, debt and ratio over time) and protocol
   history (market price, redemption price, redemption rate), sourced without a
   mandatory third-party service.
2. Options that reduce timing correlation on the private path, such as delaying
   the return to a private account, with the trade-off shown to the user.
3. Once the RFP-014 engine is integrated with RFP-013, show a position that has
   been liquidated: what was seized, the debt cleared, and anything returned.
4. Shared components and design tokens structured so the companion liquidation
   and auction interface can reuse them, consistent with the
   [Logos design system](https://github.com/logos-co/logos-design-system).
5. Notify the user, while the mini-app is running, when a position crosses the
   warning margin of Usability #6.

## ⚠ Platform Dependencies

### Hard dependencies

#### Reflexive Stablecoin Protocol (RFP-013)

This interface operates the RFP-013 program and calls it through the RFP-013
Logos core module. The program is implemented in
[logos-blockchain/lez-programs](https://github.com/logos-blockchain/lez-programs)
(`programs/stablecoin`, design in `docs/stablecoin/README.md`), and its core
module operations are being completed there. Neither is deployed on a public
testnet at the time of writing. The program's interface relevant to this RFP:

- Positions are public accounts addressed by `(owner, nonce)`, each with its own
  collateral vault. Every later operation on a position must be signed by its
  owner. There is no on-chain index of positions, so clients find them by
  deriving addresses from known owners and nonces. A nonce cannot be reused
  after a position is closed.
- Collateralisation is checked against the redemption price, not the market
  price. The price feed is used by the controller and as a staleness gate on
  minting.
- Repayment burns an exact amount and rejects overpayment, which is why
  Functionality #3 is stated as it is.

This RFP does not include changes to the RFP-013 program. Where a requirement
here cannot be met with the program as delivered, the proposal must identify it.

### Soft dependencies

#### Wallet

Signing, account selection and proving go through the Logos wallet. Today that
is the LEZ core module
([logos-execution-zone-module](https://github.com/logos-blockchain/logos-execution-zone-module)),
which Basecamp shares across mini-apps.
[LP-0021](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0021.md)
(open) specifies a Wallet Provider SDK with account selection and approval
prompts. The interface should use it once available, and proposals should state
how they move to it.

#### Price feed

The interface reads whichever price feed the RFP-013 deployment is configured
with, either [RFP-019](./RFP-019-twap-oracle.md) (on-chain TWAP) or
[RFP-020](./RFP-020-redstone-oracle-adaptor.md) (RedStone adaptor). It does not
integrate a feed of its own.

#### History

The RFP-013 program does not currently emit events, and there is no
program-level LEZ indexer yet. Soft Requirement #1 may therefore need an indexer
of the team's own, built as a Logos module, or reconstruction from transaction
history over RPC. Proposals should state which.

### Risks

#### The private path has no implementation yet

RFP-013 assigns the private account path to the client, and the RFP-013 core
module operations under review accept public signers only. This RFP is the first
implementation of that path for a long-lived position, where the owner account
cannot be discarded after one operation as it can for a swap. Proposals should
treat Functionality #5 and Privacy #2 as the core design problem and address
them first.

#### Upstream schedule

The interface cannot be tested end to end until the RFP-013 program and core
module are deployed on LEZ testnet 0.4. A phased proposal is welcome, with
designs and the core module developed against a local sequencer first.

## 👤 Recommended Team Profile

Team experienced with:

- Interfaces for CDP, lending or other leveraged DeFi protocols
- Qt/QML or comparable native desktop UI development
- Product design for financial risk (presenting liquidation risk, previews,
  confirmation flows)
- Wallet integration and client-side transaction construction
- Privacy analysis of client behaviour on public chains

## ⏱ Timeline Expectations

Estimated duration: **8–12 weeks**, excluding time spent waiting for the RFP-013
deployment on LEZ testnet 0.4.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
- [RFP-014 — Liquidation & Auction Engine](./RFP-014-liquidation-auction-engine.md)
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
