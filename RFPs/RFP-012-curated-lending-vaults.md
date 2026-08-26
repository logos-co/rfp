---
id: RFP-012
title: Curated Lending Vaults
tier: L
status: closed
dependencies:
  - id: RFP-008
    reason: The vault layer allocates deposits across the isolated lending markets delivered by RFP-008 and cannot proceed until the base protocol is live.
  - id: LP-0013
    reason: Token transfer-authority primitives are required for the vault program to custody idle balances and issue vault share tokens.
category: Applications & Integrations
---

# RFP-012 — Curated Lending Vaults

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

Build a curated vault layer on top of the isolated-market lending protocol
delivered by [RFP-008](./RFP-008-lending-borrowing-protocol.md). Each vault
accepts deposits of a single loan token, issues transferable vault share tokens
(LEZ fungible tokens), and allocates deposits across multiple Morpho-style
lending markets according to a curator-defined strategy. This is the Morpho
Vaults V2 equivalent for LEZ: a vault layer that abstracts isolated markets into
a single deposit interface for passive lenders.

Morpho Blue's isolated-market core is powerful but requires lenders to choose
individual markets manually (evaluating collateral quality, oracle reliability,
LLTV risk). Curated vaults abstract this complexity: depositors supply a single
token, the curator allocates across vetted markets, and depositors earn blended
yield. On Ethereum, the MetaMorpho vault layer absorbs the majority of the full
Morpho stack ($11.78B TVL across Blue + vaults by 2026-05, with the Blue core
itself at ~$4.9B); Coinbase's USDC lending product alone routes multi-billion
AUM through a single Steakhouse-curated MetaMorpho vault. The applying team
should ideally be the same team that delivered RFP-008, or have deep familiarity
with the deployed protocol's internals.

## 🔥 Why This Matters

The isolated-market model in RFP-008 optimises for safety and
permissionlessness, but it fragments liquidity: each market is independent, and
lenders must actively manage exposure across markets. Most passive lenders (the
largest source of TVL in any lending ecosystem) will not do this. Without
curated vaults, the protocol's TVL ceiling is limited to active, sophisticated
lenders.

Curated vaults solve this by aggregating passive capital and directing it where
it earns the best risk-adjusted yield. On Ethereum, the majority of Morpho
deposits flow through MetaMorpho vaults rather than directly into isolated
markets. For the Logos ecosystem, vaults make the lending protocol usable for
passive depositors who would not otherwise interact with raw isolated markets.
This widens the protocol's addressable market.

Multiple curators can offer different risk and yield profiles in parallel, and
depositors choose the vault that matches their preference. This avoids
concentrating risk decisions in a single governance body.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

01. **Vault creation**: anyone can create a vault by specifying the loan token,
    a vault name, and a vault symbol. The creator becomes the vault's initial
    curator. Vault creation is permissionless.
02. **Vault share tokens**: depositors receive LEZ fungible tokens representing
    their proportional share of the vault's total assets. Vault shares are
    freely transferable and composable with other LEZ programs. The
    share-to-asset exchange rate increases monotonically as interest accrues
    (absent bad debt).
03. **Deposit and withdraw**: users can deposit the vault's loan token and
    receive vault shares. Users can redeem vault shares for the underlying loan
    token, subject to available liquidity across the vault's allocated markets.
04. **Market allocation via adapters**: the vault connects to RFP-008 lending
    markets through adapters (enabled yield sources). The curator enables and
    disables adapters; an authorised allocator moves capital from the vault's
    idle balance into enabled adapters (`allocate`) and back to idle
    (`deallocate`). There is no ordered supply or withdraw queue: routing is
    adapter-based, consistent with Morpho Vaults V2.
05. **ID-based risk caps**: the curator sets absolute and relative caps keyed to
    a risk id (a collateral asset, a market, or a protocol), bounding the
    vault's maximum exposure to that risk class. Allocators may reallocate only
    within these caps. The curator designates a `liquidityAdapter` that serves
    user deposits and withdrawals, so that any liquidity supplied by the vault
    is reachable on withdrawal.
06. **Curator role**: the curator can enable or disable adapters (adding or
    removing markets from the vault's allocation set), increase caps for any
    risk id (subject to timelock) and instantly decrease them, and set a
    performance fee. The curator can transfer the curator role to another
    account. The curator cannot allocate or deallocate capital directly.
07. **Allocator role**: the curator can authorise allocator accounts that may
    allocate capital from idle assets into enabled adapters and deallocate it
    back to idle, manage the `liquidityAdapter`, and set a maximum asset growth
    rate, all within the curator's caps. Allocators cannot enable new adapters,
    change caps, or set fees.
08. **Performance fee**: a configurable percentage of yield accrued by the vault
    is directed to the curator as a performance fee. The fee is denominated in
    vault shares. Lazy fee accrual (debit on claim, or batched mint when accrued
    fees cross a threshold) is acceptable provided supply share value and
    pending-fee accounting reflect accrued-but-unclaimed fees on read.
09. **Timelock for parameter changes**: risk-increasing changes (cap increases
    for any risk id, enabling new adapters, fee adjustments, allocator
    additions, gate-contract changes) and sentinel appointment or rotation are
    subject to a configurable timelock. In Morpho Vaults V2 the timelock is
    configured per action (per selector), with no protocol-enforced minimum (it
    can be zero, the default at creation) up to an overflow-bounded maximum. Cap
    decreases by the curator or sentinel are instant and not timelocked. Any
    address may execute a timelocked action once its delay has expired. The
    applicant should define a sensible minimum timelock for risk-increasing
    actions for LEZ; this minimum is set at vault creation and is immutable.
10. **Sentinel role** (Morpho Vaults V2 semantics): the owner can appoint one or
    more sentinels whose mandate is risk reduction only. A sentinel can: (a)
    deallocate assets from any enabled adapter back to the vault's idle balance,
    (b) instantly decrease absolute or relative caps for any risk id, and (c)
    revoke any pending timelocked action submitted by the curator. A sentinel
    **cannot** introduce new risk: it cannot add adapters, increase caps,
    allocate capital into new markets, or modify fees. This is the safety
    mechanism against compromised curator keys. Reference:
    [Morpho Vaults V2 Roles & Capabilities](https://docs.morpho.org/morpho-vaults/concepts/roles/).

#### Usability

1. All vault features are exposed by extending the **core module** delivered in
   RFP-008 (the headless business-logic module that the GUI, CLI, and reference
   services all consume). The CLI and GUI built in RFP-008 are extended to
   surface the new vault features; no separate front-end is required.
2. The GUI displays per-vault: current APY, total deposits, allocated markets
   with individual utilisation, risk-id caps (absolute and relative) and
   remaining capacity, and the curator's performance fee.
3. The GUI displays per-user: vault share balance, current value in underlying
   token, and accrued yield since deposit.
4. When interacting via a private account, the SDK must handle the atomic
   deshield (deposit token + native gas) as a single indivisible user action,
   consistent with the RFP-008 privacy pattern.
5. When interacting via a private account, before each vault operation, the GUI
   must show the estimated transaction fee and confirm that the user's shielded
   balance covers both the deposit amount and fees within the single deshield
   action.
6. Vault share tokens must be usable as collateral in RFP-008 lending markets
   (subject to an oracle being available for the vault share price).

#### Reliability

1. Vault share value (assets per share) is monotonically non-decreasing absent
   bad debt in underlying markets.

2. Risk caps are enforced atomically; concurrent deposits cannot jointly exceed
   an absolute or relative cap for any risk id.

3. Withdrawal always succeeds if the total available liquidity across the
   vault's enabled adapters covers the requested amount, even if individual
   markets have insufficient liquidity (the vault draws from multiple adapters).
   When underlying markets are illiquid, in-kind redemption via flash loan
   (`forceDeallocate`) lets a depositor force liquidity rather than being locked
   in; a curator-configurable exit penalty applies, consistent with Morpho
   Vaults V2.

4. **Bad-debt realisation and socialisation.** If a market in the vault's
   allocation incurs bad debt, the loss is socialised across all remaining vault
   depositors proportionally to their share at the time the loss is realised.
   Bad debt is accounted **on touch**: the vault's exposure to a damaged market
   is repriced when any operation (deposit, withdrawal, reallocation, fee
   accrual) next reads that market's state, mirroring the lazy-accrual model of
   RFP-008. The vault must track and report realised bad debt per market.

   This is the same realisation model Morpho uses and it has a known asymmetry:
   under the adapter-based allocation model (F4, F5), a depositor who withdraws
   between the bad-debt event and the next-touch repricing of the affected
   market can draw against healthy adapters and exit at an inflated share price,
   passing a larger proportional loss to the depositors who remain. Atomic,
   cross-market repricing is not feasible without transaction-atomic price feeds
   across every allocated market. Applicants must document this asymmetry in the
   privacy and properties document (Supportability #8) and surface it in the
   GUI's risk disclosures, so depositors understand that the first interaction
   after a market loss bears a disproportionate cost.

#### Performance

1. Document the transaction size of each vault operation (deposit, withdraw,
   reallocate) on LEZ. LEZ's block size is limited and this budget may change
   during testnet.
2. A vault with up to 10 allocated markets must not exceed LEZ compute limits on
   any single deposit or withdrawal operation.

#### Supportability

1. The updated program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI; CI must be green on the default branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
4. A README addendum documents the vault features: vault creation, curator
   setup, market allocation, deposit/withdrawal flow, and sentinel
   configuration.
5. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for each new SDK feature (vault creation, deposit/withdraw, market
   allocation, curator/allocator/sentinel roles), covering the developer
   integration journey.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI additions covering the vault features.
7. Provide Figma designs or equivalent for all GUI additions to the GUI.
8. Update the privacy and anonymisation properties document delivered by RFP-008
   to cover the vault layer: what vault state and operations are visible to
   observers; what data is protected when the private account path is used for
   vault interactions; trust assumptions, specifying which guarantees are
   enforced by the on-chain program and which depend on correct client
   behaviour; and what happens if a user bypasses the expected interaction path.
   See
   [RFP-008 — Privacy Architecture](./RFP-008-lending-borrowing-protocol.md#privacy-architecture)
   for the baseline this document must align with.
9. **Audit programme.** The proposal must include a planned audit programme
   covering the vault program and the role-permission surface (curator,
   allocator, sentinel). The proposal must name at least one tier-1 audit firm
   the applicant intends to engage, include the audit budget as a line item, and
   include the audit timeline ahead of any mainnet recommendation. Audit reports
   must be published with the codebase before mainnet deployment is recommended.
   If RFP-008 and RFP-012 are delivered by the same team, the audit programmes
   may be combined in a single engagement; the proposal must make the combined
   scope explicit.

#### + Privacy

1. The GUI and SDK must support both direct public account interaction and the
   deshield→interact→reshield pattern for private account interaction with
   vaults. When a user chooses the private account path, the SDK must enforce
   the complete deshield→interact→reshield pattern; the reshield step must not
   be skippable.
2. When using the private account path, the GUI must display a pre-confirmation
   summary for each vault operation that clearly identifies what will be visible
   on-chain (deposit/withdrawal amount, vault address, vault share mint/burn,
   ephemeral intermediary account) and what will remain private (the originating
   private account, the destination of re-shielded vault shares or withdrawn
   tokens, and any link between separate interactions by the same user).
3. The ephemeral public account (account A) created during the deshield step
   must never be reused across vault operations. Each interaction from a private
   account must use a freshly generated account with no prior on-chain history.

### Soft Requirements

If possible.

#### Functionality

1. **Vault performance history**: an on-chain or indexed record of historical
   vault APY, enabling depositors to evaluate curator track records before
   depositing.

#### Reliability

1. **Formal verification of vault invariants**: (a) vault share value cannot
   decrease except through realised bad debt, (b) no risk-id cap (absolute or
   relative) is ever exceeded, (c) timelock duration cannot be shortened after
   vault creation.

### Out of Scope

- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Any feature already delivered by RFP-008

### Privacy Architecture

This RFP inherits the privacy model defined for the core lending protocol. The
same deshield, interact, reshield pattern applies to every vault operation
(deposit, withdraw, vault share transfer). See
[RFP-008 — Privacy Architecture](./RFP-008-lending-borrowing-protocol.md#privacy-architecture)
for the full interaction flow, the gas constraint, and the public/ private
observability split.

## ⚠ Platform Dependencies

This RFP is open for proposal submission. However, development is blocked until
the following is satisfied:

1. **RFP-008 is live on LEZ devnet/testnet**: this RFP extends the deployed
   protocol; it cannot proceed without the base layer.

All other platform primitives required by this RFP (including LP-0013 token
transfer authorities and an oracle provider) are hard blockers for RFP-008 and
will therefore be resolved before this RFP opens. LP-0015 (general cross-program
calls), once a blocker, is delivered. See
[RFP-008 — Platform Dependencies](./RFP-008-lending-borrowing-protocol.md#-platform-dependencies)
for details.

## 👤 Recommended Team Profile

Team experienced with:

- Morpho Blue and MetaMorpho vault architecture (or equivalent vault/aggregator
  protocols such as Yearn, ERC-4626 vaults)
- Solana or SVM program development (Anchor or native)
- DeFi risk curation and yield optimisation strategies
- Token standard design (vault share token mechanics)
- Front-end development for DeFi applications

Ideally the same team that delivered RFP-008.

## ⏱ Timeline Expectations

Estimated duration: **12 weeks** (after platform dependencies are resolved and
RFP-008 is live).

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
- [Appendix: Lending and Borrowing Ecosystem](../appendix/lending-ecosystem.md)
  — MetaMorpho curated-vault deep dive, curator landscape, Vault V2 design
- TODO: Oracle integration guide for LEZ
- TODO: SPEL framework documentation

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
