---
id: RFP-012
title: Curated Lending Vaults
tier: L
status: closed
dependencies:
  - id: RFP-008
    reason: The vault layer allocates deposits across the isolated lending markets specified in RFP-008 and cannot proceed until the base protocol is live.
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

Build a curated vault layer on top of the isolated-market lending protocol
specified in [RFP-008](./RFP-008-lending-borrowing-protocol.md). Each vault
accepts deposits of a single loan token, issues transferable vault share tokens
(LEZ fungible tokens), and allocates deposits across multiple Morpho-style
lending markets according to a curator-defined strategy. This is the Morpho
Vaults V2 equivalent for LEZ: a vault layer that abstracts isolated markets into
a single deposit interface for passive lenders.

The vault program targets testnet 0.4 (Supportability requirement #1).

Morpho Blue's isolated-market core is powerful but requires lenders to choose
individual markets manually (evaluating collateral quality, oracle reliability,
LLTV risk). Curated vaults abstract this complexity: depositors supply a single
token, the curator allocates across vetted markets, and depositors earn blended
yield. On Ethereum, the MetaMorpho vault layer absorbs the majority of the full
Morpho stack ($11.78B TVL across Blue + vaults by 2026-05, with the Blue core
itself at ~$4.9B); Coinbase's USDC lending product alone routes multi-billion
AUM through a single Steakhouse-curated MetaMorpho vault. The vault layer
extends the RFP-008 implementation, so the applying team needs deep familiarity
with its internals.

## 🔥 Why This Matters

The isolated-market model in RFP-008 optimises for safety and
permissionlessness, but it fragments liquidity: each market is independent, and
a lender supplying directly must choose markets and manage exposure across them
individually.

A curated vault aggregates deposits of a single token and allocates them across
markets within the caps its curator sets, so a depositor holds one vault share
position rather than positions in individual markets. On Ethereum, the majority
of Morpho deposits flow through MetaMorpho vaults rather than directly into
isolated markets.

The program supports any number of independently created vaults, each with its
own curator, caps and adapter set, so allocation decisions are not concentrated
in a single governance body.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

Roles follow Morpho Vaults V2 semantics. Reference:
[Morpho Vaults V2 Roles & Capabilities](https://docs.morpho.org/morpho-vaults/concepts/roles/).

##### Vault program

1. The vault program has no program-level admin or upgrade authority: once
   deployed, its code cannot be changed, and the account that deployed it holds
   no role in any vault.
2. Anyone can create a vault by specifying the loan token, a vault name and a
   vault symbol. Vault creation is permissionless.
3. The account that sends the vault creation transaction is assigned as the
   vault's owner and as its curator.
4. One account may hold more than one role in the same vault.

##### Vault shares, deposits and fees

5. **Vault share tokens**: depositors receive LEZ fungible tokens representing
   their proportional share of the vault's total assets. Vault shares are freely
   transferable and composable with other LEZ programs. The share-to-asset
   exchange rate increases monotonically as interest accrues (absent bad debt).
6. **Deposit and withdraw**: users can deposit the vault's loan token and
   receive vault shares. Users can redeem vault shares for the underlying loan
   token, subject to available liquidity across the vault's allocated markets.
7. **Performance fee**: a configurable percentage of yield accrued by the vault
   is directed to the curator as a performance fee. The fee is denominated in
   vault shares. Lazy fee accrual (debit on claim, or batched mint when accrued
   fees cross a threshold) is acceptable provided supply share value and
   pending-fee accounting reflect accrued-but-unclaimed fees on read.

##### Adapters

08. An adapter is the component through which the vault invests in a yield
    source. It exposes three operations to the vault: allocate (receive assets
    from the vault and invest them), deallocate (divest assets and return them
    to the vault) and report the current value of its positions in the loan
    token.
09. Allocate and deallocate each return the risk ids the operation affects and
    the signed change in the adapter's position.
10. Only the vault that enabled an adapter can call that adapter's allocate and
    deallocate.
11. Any program that implements the adapter interface can be enabled as an
    adapter, subject to the adapter registry if one is set.
12. The curator can set an adapter registry, subject to the timelock. While a
    registry is set, only adapters listed in it can be enabled, and setting a
    registry fails if any already-enabled adapter is not listed in it. With no
    registry set, any adapter can be enabled.
13. The `liquidityAdapter` is the enabled adapter to which deposits are
    allocated, and from which withdrawals are deallocated when the vault's idle
    balance does not cover them.
14. There is no ordered supply or withdraw queue: routing is adapter-based,
    consistent with Morpho Vaults V2.
15. The proposal includes an RFP-008 market adapter that can allocate to any
    RFP-008 market, with the market selected on each call.

##### Risk ids and caps

16. A risk id identifies a property shared by one or more allocations, such as
    an adapter, a collateral token or a specific market.
17. For each allocation, the RFP-008 market adapter returns three risk ids: one
    for the adapter, one for the market's collateral token, and one for the
    specific market.
18. For each risk id, the vault tracks the amount currently allocated against
    it.
19. The absolute cap of a risk id is the maximum amount, in loan-token units,
    that can be allocated against it. An absolute cap of zero blocks allocation
    to that id.
20. The relative cap of a risk id is the maximum allocation against it as a
    fraction of the vault's total assets. A relative cap of 100% means no
    relative limit.
21. Caps are checked on every allocation, including deposits routed to the
    `liquidityAdapter`, and an allocation that would exceed a cap fails. Caps
    are not checked on deallocation or withdrawal, so interest accrued on a
    position can take its allocation above a cap.

##### Timelock

22. Curator actions marked as subject to the timelock are submitted first and
    can be executed only once their delay has expired. Any address may execute a
    timelocked action once its delay has expired. In Morpho Vaults V2 the
    timelock is configured per action (per selector), with no protocol-enforced
    minimum (it can be zero, the default at creation) up to an overflow-bounded
    maximum. Cap decreases by the curator or sentinel are instant and not
    timelocked. The applicant should define a sensible minimum timelock for
    risk-increasing actions (at least cap increases, enabling adapters, fee
    changes, allocator additions and gate-contract changes) for LEZ; this
    minimum is set at vault creation and is immutable.

##### Owner

23. Each vault has exactly one owner.
24. The owner can appoint a new curator. The change takes effect immediately and
    is not timelocked.
25. The owner can appoint and remove sentinels. The change takes effect
    immediately and is not timelocked.
26. The owner can transfer ownership to another account. The transfer takes
    effect immediately and is not timelocked.
27. The owner cannot enable or disable adapters, change caps or fees, authorise
    or remove allocators, or allocate or deallocate capital.

##### Curator

28. Each vault has exactly one curator.
29. The curator can enable and disable adapters, adding or removing markets from
    the vault's allocation set, subject to the timelock.
30. The curator can increase absolute and relative caps for any risk id, subject
    to the timelock, and decrease them instantly.
31. The curator can set the performance fee, subject to the timelock.
32. The curator can authorise and remove allocators, subject to the timelock.
33. The curator can set gate contracts, subject to the timelock.
34. The curator can set a forced-deallocation penalty for each adapter
    (Reliability #3), up to 2% of the amount force-deallocated, subject to the
    timelock.
35. The curator can change per-action timelock durations, subject to the
    timelock. A decrease is delayed by the current timelock of the action being
    decreased. No risk-increasing action's timelock can be set below the minimum
    fixed at vault creation, and each risk-increasing action's timelock starts
    at that minimum.
36. The curator cannot allocate or deallocate capital, appoint or remove
    sentinels, or appoint a curator.

##### Allocator

37. A vault has zero or more allocators.
38. An allocator can allocate idle assets into enabled adapters and deallocate
    them back to idle, within the curator's caps.
39. An allocator can set the `liquidityAdapter` and the maximum asset growth
    rate. These changes take effect immediately and are not timelocked.
40. An allocator cannot enable or disable adapters, change caps or fees, or
    change any role.

##### Sentinel

41. A vault has zero or more sentinels, whose mandate is risk reduction only.
    This is the safety mechanism against compromised curator keys.
42. A sentinel can deallocate assets from any enabled adapter back to idle,
    instantly decrease any absolute or relative cap, and revoke any pending
    timelocked action.
43. A sentinel cannot enable adapters, increase caps, allocate capital, change
    fees, or change any role.

#### Usability

1. All vault features are exposed by extending the **core module** specified in
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
   under the adapter-based allocation model (Functionality #8 to #21), a
   depositor who withdraws between the bad-debt event and the next-touch
   repricing of the affected market can draw against healthy adapters and exit
   at an inflated share price, passing a larger proportional loss to the
   depositors who remain. Atomic, cross-market repricing is not feasible without
   transaction-atomic price feeds across every allocated market. Applicants must
   document this asymmetry in the privacy and properties document
   (Supportability #8) and surface it in the GUI's risk disclosures, so
   depositors understand that the first interaction after a market loss bears a
   disproportionate cost.

#### Performance

1. Document the transaction size of each vault operation (deposit, withdraw,
   reallocate) on LEZ. LEZ's block size is limited and this budget may change
   during testnet.
2. A vault with up to 10 allocated markets must not exceed LEZ compute limits on
   any single deposit or withdrawal operation.

#### Supportability

1. The vault program is deployed and tested on testnet 0.4.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI; CI must be green on the default branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
4. A README addendum documents the vault features: vault creation, curator
   setup, market allocation, deposit/withdrawal flow, and sentinel
   configuration.
5. Provide documentation in repo for each new SDK feature (vault creation,
   deposit/withdraw, market allocation, curator/allocator/sentinel roles),
   covering the developer integration journey.
6. Provide documentation in repo for the CLI additions, covering the vault
   features.
7. Provide Figma designs or equivalent for all GUI additions to the GUI.
8. Update the privacy and anonymisation properties document required by RFP-008
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
   include the audit timeline. Audit reports must be published with the codebase
   before any production deployment of the software. If RFP-008 and RFP-012 are
   implemented by the same team, the audit programmes may be combined in a
   single engagement; the proposal must make the combined scope explicit.

#### + Privacy

1. The GUI and SDK must support both direct public account interaction and the
   deshield→interact→reshield pattern for private account interaction with
   vaults. When a user chooses the private account path, the SDK must complete
   the full deshield→interact→reshield pattern for the operation.
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
   relative) is ever exceeded, (c) no risk-increasing action's timelock can be
   set below the minimum fixed at vault creation.

### Out of Scope

- Native stablecoin issuance (CDP-style minting against collateral)
- Governance token design and distribution
- Cross-chain liquidity or bridging
- Leveraged looping / one-click multiply products
- Any feature already specified in RFP-008
- Mainnet deployment. Mainnet deployment is not part of this RFP: whether, when
  and by whom the software is deployed is a matter for those who deploy it.
- The identity of persons holding the owner, curator, allocator and sentinel
  roles, and the exercise of vault-specific governance and operational powers in
  any deployment, are matters for the relevant independent deployer or operator
  and are outside the scope of this RFP.

### Privacy Architecture

This RFP inherits the privacy model defined for the core lending protocol. The
same deshield, interact, reshield pattern applies to every vault operation
(deposit, withdraw, vault share transfer). See
[RFP-008 — Privacy Architecture](./RFP-008-lending-borrowing-protocol.md#privacy-architecture)
for the full interaction flow, the gas constraint, and the public/ private
observability split.

## ⚠ Platform Dependencies

This RFP is open for proposal submission. However, implementation of the vault
layer depends on the availability of the RFP-008 base protocol on testnet 0.4.
The underlying platform dependencies are described in
[RFP-008 Platform Dependencies](./RFP-008-lending-borrowing-protocol.md#-platform-dependencies)
and must be resolved to the extent necessary for integration.

## 👤 Recommended Team Profile

Team experienced with:

- Morpho Blue and MetaMorpho vault architecture (or equivalent vault/aggregator
  protocols such as Yearn, ERC-4626 vaults)
- Solana or SVM program development (Anchor or native)
- DeFi risk curation and yield optimisation strategies
- Token standard design (vault share token mechanics)
- Front-end development for DeFi applications
- The internals of the RFP-008 implementation that the vault layer extends

## ⏱ Timeline Expectations

Estimated duration: **12 weeks** (after platform dependencies are resolved and
RFP-008 is live).

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
- [Appendix: Lending and Borrowing Ecosystem](../appendix/lending-ecosystem.md)
  — MetaMorpho curated-vault deep dive, curator landscape, Vault V2 design
- [RFP-020 RedStone Off-Chain Oracle Adaptor](./RFP-020-redstone-oracle-adaptor.md)
  and [RFP-019 On-Chain TWAP Oracle](./RFP-019-twap-oracle.md): oracle sources
  on LEZ
- [SPEL framework](https://github.com/logos-co/spel)

Except for the dependencies identified under Platform Dependencies, the
Resources listed above, and any third-party material referred to elsewhere in
this specification, are for informational purposes only and are intended solely
as guidance regarding tools or other artefacts that may be relevant to preparing
a Submission under the Program. They may include, or may refer to earlier
published specifications, some of which may have received Submissions. Logos
does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
