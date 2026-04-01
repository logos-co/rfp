---
id: RFP-017
title: Privacy-Preserving Token Vesting
tier: L
funding: $XXXXX
status: draft
category: Applications & Integrations
---


# RFP-017: Privacy-Preserving Token Vesting

## 🧭 Overview

Build a token vesting program on LEZ that locks project, team, or
investor tokens against configurable time-based schedules (cliff,
linear, or milestone-based). Recipients can claim vested tokens
directly from a public account, or via a deshield→claim→re-shield
pattern for private account interaction: when the private path is
used, the claim is executed from a fresh, single-use public account,
and the output is immediately re-shielded to the recipient's private
account, making the link between the claim and the recipient's
private identity untraceable on-chain.

Token vesting is a prerequisite for credible token launches. Every
protocol that issues a token must lock some allocation for founders,
early contributors, or ecosystem reserves. The team building this
should have experience with SVM program development and DeFi token
mechanics. Familiarity with the optional public/private account
interaction pattern (see
[RFP-008](./RFP-008-lending-borrowing-protocol.md)) is strongly
recommended.

## 🔥 Why This Matters

On transparent chains, vesting contracts (Streamflow, Sablier,
Team.Finance) are surveillance tools: every cliff unlock, every
linear release, and every claim is publicly linkable to a beneficiary
address, exposing token unlock schedules to the market and enabling
adversarial trading against known unlock events. A Solana founder's
vesting cliff is trivially queryable on-chain; traders index these
events and position against them before the beneficiary can act.

On LEZ, the optional deshield→claim→re-shield pattern makes claims
unlinkable to the recipient's private identity. Observers see that
tokens were claimed from a vesting schedule; they cannot determine
which private account holds those tokens or trace subsequent
movements. This makes LEZ the only environment where teams can vest
tokens without creating a public unlock calendar for adversarial
front-running.

Vesting is also a composition primitive. A launchpad (see
[RFP-016](./RFP-016-lbp-launchpad.md)) or investor round on LEZ
needs vesting to be in place before tokens can be distributed to
contributors and early backers. This RFP is a direct prerequisite
for a credible token economy on LEZ.

## 🏗 Design Rationale

### Three schedule types: cliff+linear, fully linear, milestone

These three types cover the full range of real-world vesting
relationships without unnecessary complexity.

**Cliff + linear** is the industry standard for team and investor
allocations. The cliff (commonly 1 year) prevents immediate selling
at TGE and signals that insiders have a long-term commitment. Linear
release after the cliff aligns incentives continuously over the
remaining period; recipients have ongoing reason to contribute.
This pattern is used in virtually every serious protocol's team
allocation.

**Fully linear** (no cliff) is appropriate for ongoing contributors
(advisors, contractors) where there is no fixed start date requiring
a binary lock-up, or for grant programmes where disbursement should
begin immediately.

**Milestone-based** serves deliverable-oriented relationships:
grants, partnerships, or service agreements where time-based vesting
is economically incorrect. Unlocking tokens when a milestone is
achieved rather than when a date passes better reflects the nature
of the obligation. Importantly, milestone authority stays with the
creator, not an on-chain oracle; this is intentional: the
conditions for milestone completion are often qualitative and cannot
be reliably evaluated on-chain.

### Beneficiary as public account address

A ZK commitment-based beneficiary (where the schedule creator
commits to a hash of the beneficiary's key and the recipient proves
knowledge of the preimage at claim time) would hide who is in the
schedule. This is excluded as a hard requirement for two reasons.

First, in virtually all vesting contexts the parties already know
each other: a project knows its co-founders, investors, and grant
recipients. Hiding the beneficiary's address from on-chain observers
does not change the information available to the other party in the
relationship.

Second, the infrastructure required (a ZK membership proof
compatible with the LEZ account model) overlaps with LP-0003, which
is itself a draft prize not yet delivered. Building on an undelivered
dependency would block this RFP.

The principal privacy concern (adversarial trading against known
unlock events) is addressed by the re-shield claim pattern. Even
if an observer knows that a specific address holds a vesting
schedule, they cannot track where claimed tokens go or link
subsequent on-chain activity to that address. A future RFP or
upgrade can introduce commitment-based beneficiaries once LP-0003
is complete.

### Cancellation semantics

Unvested tokens return to the creator on cancellation; already-
vested but unclaimed tokens remain claimable by the beneficiary.
This reflects the economic reality of the relationship: vested
tokens are earned compensation that cannot be revoked. Unvested
tokens represent future commitment not yet earned, and returning
them to the creator on early termination is the industry standard,
as used by Streamflow, Sablier, and Team.Finance alike. Burning
unvested tokens would destroy value unnecessarily and make the
program harder to adopt.

### Transferable positions as a creation-time flag

Whether a vesting position is transferable to a new beneficiary
address is set at creation and cannot be changed afterwards. For
employment vesting, the incentive is personal; the position should
be non-transferable so the token lock applies specifically to the
individual. For investor positions, secondary transfers may be
needed (assignment of rights, secondary market sales of locked
tokens). Allowing this to be configured at creation but frozen
afterwards avoids mid-schedule ambiguity: all parties know the
rules when the schedule is created and cannot change them
unilaterally later.

### Batch creation

Token launches routinely create 10–50 vesting schedules
simultaneously (co-founders, advisors, early contributors, investor
tranches). Creating them one-by-one is gas-expensive and
error-prone. Batch creation is a practical necessity, not a
nice-to-have, for any real-world launch. The batch size is bounded
by the LEZ transaction size limit, which the implementation must
document.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a vesting program on LEZ that locks tokens in escrow
   against a configured schedule. The program must support:
   - **Cliff + linear vesting**: tokens become claimable in a lump
     sum at the cliff date, then linearly over the remaining
     duration (e.g., 1-year cliff, 3-year linear release).
   - **Fully linear vesting**: no cliff; tokens accrue continuously
     from the start date.
   - **Milestone-based vesting**: the schedule creator explicitly
     signals completion of discrete tranches; each signal unlocks
     a fixed token amount for the beneficiary to claim.
2. The schedule creator specifies the beneficiary as a public
   account address at schedule creation. Recipients can claim
   vested tokens directly from that public account, or via a
   deshield→claim→re-shield flow: a gas-only deshield funds a
   fresh single-use public account that submits the claim and
   immediately re-shields the output to the recipient's private
   account. The schedule creator does not learn the recipient's
   private account address from this interaction.
3. The schedule creator can cancel an active schedule at any time,
   returning the unclaimed, unvested token balance to the creator's
   account. Tokens already vested but not yet claimed remain
   claimable by the beneficiary after cancellation.
4. The schedule creator can configure whether the vesting position
   (beneficiary rights) is transferable to a new public account
   address. Transferability must be set at schedule creation and
   cannot be changed afterwards.
5. Multi-recipient batch creation: a creator can set up vesting
   schedules for multiple recipients in a single operation, up to
   the LEZ transaction size limit. Document the maximum number of
   recipients achievable in one transaction.
6. The program emits a log event (using the LEZ event/log mechanism;
   see
   [LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md))
   for each state transition: schedule creation, claim,
   cancellation, milestone signal, and beneficiary transfer. Event
   schema must be documented.

#### Usability

1. Provide an SDK that can be used to build Logos modules
   interacting with the vesting program. The SDK must expose the
   full lifecycle: create schedule, query claimable amount, claim,
   cancel, signal milestone, and transfer beneficiary. The SDK must
   support both direct public account interaction and the
   deshield→claim→re-shield pattern for private account
   interaction. When the private account path is used, the SDK must
   handle the gas-only deshield as a single indivisible action.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo. The mini-app must cover at minimum:
   - **Recipient view**: current vesting position, total locked,
     amount claimable now, next unlock event, claim action.
   - **Creator view**: create a new schedule (all parameters),
     list active schedules, cancel a schedule, signal a milestone.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations: create schedule, claim,
   cancel, signal milestone, and query claimable amount.
4. The mini-app must display a pre-claim confirmation summary:
   claimable amount and estimated transaction fee. When using the
   private account path, it must also confirm that the recipient's
   shielded balance covers the gas-only deshield; a clear,
   actionable error must be shown if the balance is insufficient.
5. When using the private account path, the mini-app must display
   a privacy disclosure before each claim, identifying what will
   be visible on-chain (claim amount, ephemeral public account
   address, vesting schedule program address) and what will not be
   traceable (the recipient's private account and the subsequent
   destination of claimed tokens).
6. Provide an IDL for the vesting program using the
   [SPEL framework](https://github.com/logos-co/spel).
7. Failed claims must return clear, actionable error messages,
   including the case where nothing is yet claimable (with the
   next unlock time displayed).

#### Reliability

1. A claim is atomic: a failed or rejected claim does not consume
   vested tokens and leaves the claimant able to retry without
   loss.
2. A schedule cancellation is atomic: partial failure leaves the
   schedule in its prior state, not an undefined one.
3. Concurrent claims against independent schedules must not corrupt
   shared program state.
4. Milestone signalling is idempotent per milestone index:
   signalling the same milestone twice must be rejected with a
   deterministic error and must not double-unlock tokens.

#### Performance

1. A single claim completes within one LEZ transaction.
2. Document the compute unit (CU) cost of each operation: create
   schedule, claim, cancel, signal milestone, transfer beneficiary.
   Note the LEZ testnet version against which measurements were
   taken, as the per-transaction compute budget may change.

#### Supportability

Proposals must include separate milestones for testnet 0.2, testnet 0.3,
and mainnet deployment.

1. The vesting program is deployed and tested on LEZ testnet 0.2.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI. CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for creating a
   schedule, claiming, and cancelling via CLI and mini-app.
5. The program is updated and verified on LEZ testnet 0.3.
6. The program is deployed to LEZ mainnet.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→claim→re-shield pattern for private
   account interaction. When a user chooses the private account
   path, the SDK must enforce the complete deshield→claim→re-shield
   pattern; the re-shield step must not be skippable.
2. When using the private account path, the mini-app must display
   the pre-confirmation privacy disclosure described in the
   Usability section before each claim, identifying what is visible
   on-chain (claim amount, ephemeral intermediary account) and what
   remains private (the originating private account, the
   destination of re-shielded tokens, and any link between separate
   claims by the same recipient).
3. When using the private account path, the SDK must validate that
   the re-shield target is a private (shielded) account before
   submitting the transaction, and reject with an explicit error if
   it is not.
4. The ephemeral public account created during the deshield step
   must never be reused across operations. Each claim from a
   private account must use a freshly generated account with no
   prior on-chain history.

### Privacy Architecture

Vesting schedule parameters (token, total amount, schedule type,
cliff date, end date, beneficiary public address) are public
on-chain state. This is a deliberate architectural choice: public
state enables permissionless observability of schedule terms and
composability with other programs.

User privacy is optionally enforced at the UX layer. The mini-app
and SDK support both direct public account interaction and private
account interaction via the deshield→claim→re-shield pattern. When
users opt to interact from a private account, the SDK must enforce
the complete pattern as described below.

#### Direct private account receipt (preferred if supported)

If LEZ supports program-initiated shielding (that is, a program can
directly credit a beneficiary's shielded balance without routing
through a public intermediary), this is the preferred claim design.
The beneficiary's private account address would be registered in the
vesting schedule at creation; at claim time the program shields the
vested amount directly into that address. The beneficiary never needs
to expose a public account, and all subsequent token usage remains
entirely within the private sphere via the normal
deshield→action→re-shield flow.

The implementer must investigate whether LEZ's shielding primitive
can be invoked by a program (not only by the account owner). If it
can, the direct shielding path must be used as the primary claim
flow. If it cannot, fall back to the ephemeral public account pattern
described below.

#### Ephemeral public account fallback

Where program-initiated shielding is not supported, the vesting
program can only release tokens to public token accounts; it cannot
write directly to a shielded balance. Re-shielding is then a
separate operation performed by the account owner after receiving
the tokens. This is the same fundamental constraint as the lending
protocol.

#### Interaction flow

For every claim from a private account:

1. The recipient initiates the action from their private account.
   The SDK deshields a small amount of native token for gas to a
   **fresh, single-use** public account (account A) with no prior
   on-chain history. Unlike the DEX, the recipient does not deshield
   any project token; the vested tokens come *from* the program,
   not from the recipient.
2. Account A submits the claim to the vesting program. The program
   releases the vested amount to account A's token account.
3. Account A re-shields the claimed tokens to the recipient's
   private account. Account A is never reused.

> **Gas:** Gas must come exclusively from the deshield in step 1.
> Funding account A from any external source (such as a CEX
> withdrawal or a known wallet) creates an on-chain link to an
> existing identity and breaks the privacy guarantee. The SDK must
> make this impossible.

#### What is public (observable on-chain)

- All vesting schedule parameters: token, total amount, schedule
  type, cliff date, end date, beneficiary public address.
- All claims: amount claimed, the ephemeral intermediary account
  address, and timestamp.
- Cancellations and milestone signals.

#### What is private (when using the private account path)

- Which private account the claimed tokens are re-shielded into.
- Any link between multiple claims by the same recipient (no
  on-chain linkability across ephemeral accounts).
- Subsequent movements of claimed tokens after re-shielding.

### Soft Requirements

- The creator can nominate a separate cancellation authority
  distinct from the creator wallet (useful when vesting is managed
  by a multisig or DAO).

## ⚠ Platform Dependencies

This RFP remains in **draft** until the dependencies below are
resolved. LEZ has similar programming capabilities to Solana but
several primitives required by a vesting protocol are not yet
available.

### Hard blockers

These must be available on LEZ before this RFP can open.

#### On-chain clock / timestamp

LEZ does not yet have on-chain block time. Vesting schedules are
fundamentally time-based: the cliff date, linear accrual rate, and
total vested amount at any moment are all functions of elapsed time.
Without a reliable on-chain timestamp, the program cannot determine
how much of a schedule has vested, making cliff + linear and fully
linear schedule types impossible to implement correctly.
Milestone-based vesting does not require an on-chain timestamp and
is unaffected by this blocker, but it represents only a subset of
the required functionality.

#### General cross-program calls (LP-0015)

LEZ uses a tail-call execution model with no return. A claim
operation must: (1) verify the schedule and compute the claimable
amount, (2) call the token program to transfer the vested tokens to
the beneficiary, and then (3) update the schedule state (recording
the claimed amount and timestamp). Without general cross-program
calls, step 3 cannot execute after step 2: any account could call
the update entrypoint directly, bypassing the token transfer.

[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this. This
prize is currently **open**.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission (LP-0012)

Off-chain dashboards and notification services need to react to
vesting events (schedule created, cliff reached, tokens claimed,
schedule cancelled). Without structured events, these services must
poll all accounts, which is expensive and unreliable.

[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently
**open**.

## 👤 Recommended Team Profile

Team experienced with:

- Solana or SVM program development (Anchor or native)
- Token program and account model (ATAs, PDAs, token accounts)
- DeFi smart contract security and testing
- Front-end development for wallet/DeFi applications
- Familiarity with privacy interaction patterns (shielded accounts,
  deshield/re-shield, see RFP-008)

## ⏱ Timeline Expectations

Estimated duration: **10–12 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [RFP-008: Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (reference for the optional public/private account interaction
  pattern)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
- [LP-0013: Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
- [Streamflow Finance (vesting reference)](https://streamflow.finance/)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
