---
id: RFP-017
title: Privacy-Preserving Token Vesting
tier: L
status: closed
category: Applications & Integrations
dependencies: []
---

# RFP-017: Privacy-Preserving Token Vesting

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

Build a token vesting program on LEZ that locks project, team, or investor
tokens against configurable time-based schedules (cliff, linear, or
milestone-based). Recipients can claim vested tokens directly to a public
account or to a private account. When claimed to a private account, the
destination account and subsequent token movements are not recorded in public
on-chain state.

The vesting LEZ program targets LEZ testnet 0.4 (Supportability requirement #1).

A token vesting program holds token allocations in escrow and releases them to
beneficiaries according to a configured schedule. Vesting is commonly applied to
founder, contributor, investor, and ecosystem allocations. Tokenomist recorded
$97.43B in token vesting unlocks during 2025, and Magna reached $60B peak TVL
before its acquisition by Kraken. The vesting protocols surveyed record schedule
terms and claims in public on-chain state, and none of them offers claims to a
private account (see
[Appendix: Token Vesting Ecosystem](../appendix/token-vesting-ecosystem.md)).

## 🔥 Why This Matters

On transparent chains, vesting contracts (Streamflow, Sablier, Team.Finance)
record every cliff unlock, linear release, and claim in public on-chain state,
linked to the beneficiary address. Unlock schedules are therefore readable in
advance, and analytics platforms such as Tokenomist index upcoming unlock
events.

On LEZ, recipients can claim vested tokens directly to a private account. The
claim event (amount and timestamp) is recorded in public on-chain state; the
destination private account and subsequent movements of the claimed tokens are
not. Schedule terms, including unlock dates and the beneficiary address, remain
public (see [Privacy Architecture](#privacy-architecture)).

Vesting can also be composed with other programs. A launchpad (see
[RFP-016](./RFP-016-lbp-launchpad.md)) or an investor allocation on LEZ can call
the vesting program to release distributed tokens over a schedule.

## 🏗 Design Rationale

### Three schedule types: cliff+linear, fully linear, milestone

The program supports three release patterns, which differ in what triggers a
release.

**Cliff + linear** releases nothing before the cliff date. At the cliff, the
amount accrued since the start date becomes claimable as a lump sum; the
remainder then accrues linearly until the end date. This pattern is commonly
used for team and investor allocations.

**Fully linear** (no cliff) accrues tokens continuously from the start date to
the end date, so the claimable amount starts increasing immediately.

**Milestone-based** releases a fixed amount each time completion of a tranche is
signalled, independently of elapsed time. Completion is signalled by the
schedule creator (or a nominated milestone authority; see Soft Requirements)
rather than evaluated by an on-chain oracle, because milestone conditions are
often qualitative and cannot be evaluated from on-chain data.

### Cancellation semantics

Whether a schedule is cancelable is set at creation and defaults to cancelable.
A cancelable schedule can be converted to non-cancelable (irreversible); the
reverse is not permitted. This lets a creator remove its own ability to cancel
after the schedule is created. Sablier offers the same one-way conversion;
Hedgey separates revocable and non-revocable schedules into two products (see
[Appendix: Cancellation and Revocation](../appendix/token-vesting-ecosystem.md#cancellation-and-revocation)).

When a cancelable schedule is cancelled, the program returns unvested tokens to
the creator, and tokens already vested but not yet claimed remain claimable by
the beneficiary. Cancellation therefore affects only the portion of the schedule
that has not vested at the time of cancellation. This matches the cancellation
behaviour of every surveyed protocol that supports cancellation (Streamflow,
Sablier, Team.Finance, and others). The program does not burn unvested tokens on
cancellation.

### Transferable positions as a creation-time flag

Whether a vesting position is transferable to a new beneficiary address is set
at creation and cannot be changed afterwards. A non-transferable position can
only be claimed by the beneficiary named at creation; a transferable position
can be reassigned to a new beneficiary address. Because the flag is fixed at
creation, neither party can change a schedule's transfer rules after it is
created.

### Batch creation

A single allocation event often involves many recipients (for example
co-founders, advisors, contributors, and investor tranches). Batch creation lets
a creator set up schedules for multiple recipients in one operation instead of
one transaction per recipient. The batch size is bounded by the LEZ transaction
size limit, which the implementation must document.

### Fee structure

This RFP does not specify a fee rate. Of the 10 vesting protocols surveyed, 7
charge nothing beyond gas (see
[Appendix: Fee Structures](../appendix/token-vesting-ecosystem.md#fee-structures)).
Proposals must specify:

1. **Who pays:** whether fees (if any) are charged to the schedule creator, the
   beneficiary at claim time, or both.
2. **When fees are collected:** at schedule creation, at each claim event, at
   cancellation, or a combination.
3. **Fee rate:** the exact rate or range, including whether a
   governance-activatable fee switch is included for future adjustment.
4. **Where fees are routed:** the destination account or mechanism (protocol
   treasury, burn, staking reward pool, etc.).

Any fee, treasury or governance mechanism is a matter for whoever deploys the
software. An optional broker fee for third-party integrators (launchpads,
distribution platforms) may be included but is not required.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a vesting program on LEZ that locks tokens in escrow against a
   configured schedule. The program must support:
   - **Cliff + linear vesting**: tokens become claimable in a lump sum at the
     cliff date, then linearly over the remaining duration (e.g., 1-year cliff,
     3-year linear release).
   - **Fully linear vesting**: no cliff; tokens accrue continuously from the
     start date.
   - **Milestone-based vesting**: the schedule creator explicitly signals
     completion of discrete tranches; each signal unlocks a fixed token amount
     for the beneficiary to claim.
2. The schedule creator specifies the beneficiary at schedule creation.
   Recipients can claim vested tokens to a public account or to a private
   account. The schedule creator does not learn the recipient's private account
   address from the claim interaction.
3. The schedule creator sets whether the schedule is cancelable at creation time
   (default: cancelable). A cancelable schedule can be converted to
   non-cancelable by the creator; this conversion is irreversible. When a
   cancelable schedule is cancelled, the unclaimed, unvested token balance
   returns to the creator's account. Tokens already vested but not yet claimed
   remain claimable by the beneficiary after cancellation.
4. The schedule creator can configure whether the vesting position (beneficiary
   rights) is transferable to a new beneficiary. Transferability must be set at
   schedule creation and cannot be changed afterwards.
5. Multi-recipient batch creation: a creator can set up vesting schedules for
   multiple recipients in a single operation, up to the LEZ transaction size
   limit. Document the maximum number of recipients achievable in one
   transaction.
6. The program emits a log event (using the LEZ event/log mechanism; see
   [LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md))
   for each state transition: schedule creation, claim, cancellation, milestone
   signal, and beneficiary transfer. Event schema must be documented.

#### Usability

1. Provide an SDK that can be used to build Logos modules interacting with the
   vesting program. The SDK must expose the full lifecycle: create schedule,
   query claimable amount, claim, cancel, signal milestone, and transfer
   beneficiary. The SDK must support claiming to both public and private
   accounts.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. The mini-app must
   cover at minimum:
   - **Recipient view**: current vesting position, total locked, amount
     claimable now, next unlock event, claim action.
   - **Creator view**: create a new schedule (all parameters), list active
     schedules, cancel a schedule, signal a milestone.
3. Provide a CLI that covers core functionality of the program. The CLI may have
   fewer features than the GUI mini-app but must support all essential
   operations: create schedule, claim, cancel, signal milestone, and query
   claimable amount.
4. The mini-app must display a pre-claim confirmation summary: claimable amount
   and estimated transaction fee. When using the private account path, it must
   confirm that the beneficiary account has sufficient gas to cover the claim; a
   clear, actionable error must be shown if the balance is insufficient.
5. When using the private account path, the mini-app must display a privacy
   disclosure before each claim, identifying what will be visible on-chain
   (claim amount, beneficiary address, vesting schedule address) and what will
   not be recorded in public on-chain state (the destination private account and
   subsequent movements).
6. Provide an IDL for the vesting program using the
   [SPEL framework](https://github.com/logos-co/spel).
7. Failed claims must return clear, actionable error messages, including the
   case where nothing is yet claimable (with the next unlock time displayed).

#### Reliability

1. A claim is atomic: a failed or rejected claim does not consume vested tokens
   and leaves the claimant able to retry without loss.
2. A schedule cancellation is atomic: partial failure leaves the schedule in its
   prior state, not an undefined one.
3. Concurrent claims against independent schedules must not corrupt shared
   program state.
4. Milestone signalling is idempotent per milestone index: signalling the same
   milestone twice must be rejected with a deterministic error and must not
   double-unlock tokens.

#### Performance

1. A single claim completes within one LEZ transaction.
2. Document the compute unit (CU) cost of each operation: create schedule,
   claim, cancel, signal milestone, transfer beneficiary. Measurements are taken
   on LEZ testnet 0.4.

#### Supportability

1. The team deploys and tests the vesting program on LEZ testnet 0.4.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI. CI must be green on the default branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
4. A README documents end-to-end usage: deployment steps, program addresses, and
   step-by-step instructions for creating a schedule, claiming, and cancelling
   via CLI and mini-app.

#### + Privacy

1. The mini-app and SDK must support claiming to both public and private
   accounts.
2. When using the private account path, the mini-app must display the
   pre-confirmation privacy disclosure described in the Usability section before
   each claim, identifying what is visible on-chain (claim amount, beneficiary
   address, vesting schedule address) and what is not recorded in public
   on-chain state (the destination private account and subsequent movements of
   claimed tokens).
3. When using the private account path, the SDK must validate that the claim
   target is a private (shielded) account before submitting the transaction, and
   reject with an explicit error if it is not.

### Privacy Architecture

Vesting schedule parameters (token, total amount, schedule type, cliff date, end
date, beneficiary address) are public on-chain state. This is a deliberate
architectural choice: public state enables permissionless observability of
schedule terms and composability with other programs.

Privacy is enforced at the claim layer. LEZ programs can credit tokens directly
to a private account without routing through a public intermediary: any program
may increase any account's balance, including foreign private accounts. This
means the vesting program can release tokens straight into the beneficiary's
private account at claim time.

#### Claim to public or private account

Recipients choose at claim time whether tokens go to a public account or a
private account.

When claiming to a **public account**, the recipient address, the amount, and
subsequent token movements are recorded in public on-chain state, as with
vesting claims on transparent chains.

When claiming to a **private account**, the vesting program credits the
beneficiary's private account directly. The claim event (amount and timestamp)
is recorded in public on-chain state; the destination private account and
subsequent movements of the claimed tokens are not.

#### What is public (observable on-chain)

- All vesting schedule parameters: token, total amount, schedule type, cliff
  date, end date, beneficiary address.
- All claim events: amount claimed and timestamp.
- Cancellations and milestone signals.

#### What is private (when claiming to a private account)

- Which private account holds the claimed tokens.
- Subsequent movements of claimed tokens.

#### Privacy limitation

The beneficiary address is permanently and publicly associated with the vesting
schedule from the moment it is created. Every claim from the schedule is linked
to that address in public on-chain state. The privacy property covers post-claim
token movements only: the destination private account and subsequent activity
are not recorded in public on-chain state, but the beneficiary's association
with the vesting position is. Keeping the beneficiary address itself out of
public on-chain state would require commitment-based beneficiary registration
(depending on LP-0003), which is outside the scope of this RFP.

### Soft Requirements

- The creator can nominate a separate cancellation authority distinct from the
  creator wallet (for example, a multisig or DAO account).
- The creator can nominate a separate milestone authority distinct from the
  creator wallet, so that milestone signalling can be held by a party other than
  the schedule creator.

### Out of Scope

Mainnet deployment is not part of this RFP: whether, when and by whom the
software is deployed is a matter for those who deploy it.

## 👤 Recommended Team Profile

Team experienced with:

- Solana or SVM program development (Anchor or native)
- Token program and account model (ATAs, PDAs, token accounts)
- DeFi smart contract security and testing
- Front-end development for wallet/DeFi applications
- Familiarity with privacy interaction patterns (public and private accounts,
  see RFP-008)

## ⏱ Timeline Expectations

Estimated duration: **10–12 weeks**

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [RFP-008: Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (reference for the optional public/private account interaction pattern)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
- [LP-0013: Token program improvements: authorities](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
- [Streamflow Finance (vesting reference)](https://streamflow.finance/)
- [Appendix: Token Vesting Ecosystem](../appendix/token-vesting-ecosystem.md)
  (survey of existing vesting protocols, scale data, and fee structures)

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
