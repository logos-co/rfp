---
id: RFP-017
title: Privacy-Preserving Token Vesting
tier: L
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-017: Privacy-Preserving Token Vesting

## 🧭 Overview

Build a token vesting program on LEZ that locks project, team, or
investor tokens against configurable time-based schedules (cliff,
linear, or milestone-based). Recipients can claim vested tokens
directly to a public account or to a private account. When claimed
to a private account, the destination and all subsequent token
movements are hidden from on-chain observers.

Token vesting is a prerequisite for credible token launches. Every
protocol that issues a token must lock some allocation for founders,
early contributors, or ecosystem reserves. The scale of this market
is substantial: Tokenomist documented $97.43B in token vesting
unlocks during 2025 alone, and Magna (the largest vesting platform)
reached $60B peak TVL before being acquired by Kraken. Despite this
scale, leading protocols (Sablier, Hedgey, Jupiter Lock) charge
zero fees, and no existing protocol offers any privacy features
for vesting (see
[Appendix: Token Vesting Ecosystem](../appendix/token-vesting-ecosystem.md)).

## 🔥 Why This Matters

On transparent chains, vesting contracts (Streamflow, Sablier,
Team.Finance) are surveillance tools: every cliff unlock, every
linear release, and every claim is publicly linkable to a beneficiary
address, exposing token unlock schedules to the market and enabling
adversarial trading against known unlock events. With $97.43B in
token unlocks during 2025 tracked by public analytics platforms
like Tokenomist, traders routinely index upcoming unlock events and
position against them before the beneficiary can act. The April 2025
MANTRA ($OM) collapse (90% in hours, from over $6 to under $0.50,
wiping over $5B in market cap), while not solely caused by
front-running, illustrates the fragility of transparent unlock
mechanics at scale.

On LEZ, recipients can claim vested tokens directly to a private
account. Observers see that a claim occurred; they cannot determine
which private account holds the tokens or trace subsequent
movements. This makes LEZ the only environment where teams can
vest tokens without creating a public unlock calendar for
adversarial front-running.

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

### Cancellation semantics

Whether a schedule is cancelable is set at creation and defaults
to cancelable. A cancelable schedule can be converted to
non-cancelable (irreversible); the reverse is not permitted. This
one-way conversion serves as a credible commitment mechanism:
a project can issue vesting to investors and later make it
irrevocable as a trust signal, following the pattern established
by Sablier and Hedgey (see
[Appendix: Cancellation and Revocation](../appendix/token-vesting-ecosystem.md#cancellation-and-revocation)).

When a cancelable schedule is cancelled, unvested tokens return
to the creator; already-vested but unclaimed tokens remain
claimable by the beneficiary. This reflects the economic reality
of the relationship: vested tokens are earned compensation that
cannot be revoked. Unvested tokens represent future commitment
not yet earned, and returning them to the creator on early
termination is the universal invariant across every protocol in
the ecosystem (Streamflow, Sablier, Team.Finance, and others
alike). Burning unvested tokens would destroy value unnecessarily
and make the program harder to adopt.

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

Token launches routinely create vesting schedules for many
recipients simultaneously (co-founders, advisors, early
contributors, investor tranches). Creating them one by one is
gas-expensive and error-prone. Batch creation is a practical necessity, not a
nice-to-have, for any real-world launch. The batch size is bounded
by the LEZ transaction size limit, which the implementation must
document.

### Fee structure

This RFP does not mandate a specific fee rate. The ecosystem trend
is decisively toward zero protocol fees: 7 of 10 vesting protocols
surveyed charge nothing beyond gas (see
[Appendix: Fee Structures](../appendix/token-vesting-ecosystem.md#fee-structures)).
Proposals must specify:

1. **Who pays:** whether fees (if any) are charged to the schedule
   creator, the beneficiary at claim time, or both.
2. **When fees are collected:** at schedule creation, at each claim
   event, at cancellation, or a combination.
3. **Fee rate:** the exact rate or range, including whether a
   governance-activatable fee switch is included for future
   adjustment.
4. **Where fees are routed:** the destination account or mechanism
   (protocol treasury, burn, staking reward pool, etc.).

A governance-activatable fee switch with an initial zero rate
is the recommended baseline. An optional broker fee for
third-party integrators (launchpads, distribution platforms)
may be included but is not required.

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
2. The schedule creator specifies the beneficiary at schedule
   creation. Recipients can claim vested tokens to a public account
   or to a private account. The schedule creator does not learn the
   recipient's private account address from the claim interaction.
3. The schedule creator sets whether the schedule is cancelable at
   creation time (default: cancelable). A cancelable schedule can
   be converted to non-cancelable by the creator; this conversion
   is irreversible. When a cancelable schedule is cancelled, the
   unclaimed, unvested token balance returns to the creator's
   account. Tokens already vested but not yet claimed remain
   claimable by the beneficiary after cancellation.
4. The schedule creator can configure whether the vesting position
   (beneficiary rights) is transferable to a new beneficiary.
   Transferability must be set at schedule creation and cannot be
   changed afterwards.
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
   support claiming to both public and private accounts.
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
   private account path, it must confirm that the beneficiary
   account has sufficient gas to cover the claim; a clear,
   actionable error must be shown if the balance is insufficient.
5. When using the private account path, the mini-app must display
   a privacy disclosure before each claim, identifying what will
   be visible on-chain (claim amount, beneficiary address, vesting
   schedule address) and what will not be traceable (the
   destination private account and subsequent movements).
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

1. The mini-app and SDK must support claiming to both public and
   private accounts.
2. When using the private account path, the mini-app must display
   the pre-confirmation privacy disclosure described in the
   Usability section before each claim, identifying what is visible
   on-chain (claim amount, beneficiary address, vesting schedule
   address) and what remains private (the destination private
   account and subsequent movements of claimed tokens).
3. When using the private account path, the SDK must validate that
   the claim target is a private (shielded) account before
   submitting the transaction, and reject with an explicit error if
   it is not.

### Privacy Architecture

Vesting schedule parameters (token, total amount, schedule type,
cliff date, end date, beneficiary address) are public on-chain
state. This is a deliberate architectural choice: public state
enables permissionless observability of schedule terms and
composability with other programs.

Privacy is enforced at the claim layer. LEZ programs can credit
tokens directly to a private account without routing through a
public intermediary: any program may increase any account's
balance, including foreign private accounts. This means the
vesting program can release tokens straight into the
beneficiary's private account at claim time.

#### Claim to public or private account

Recipients choose at claim time whether tokens go to a public
account or a private account.

When claiming to a **public account**, the claim is fully
transparent: observers see the recipient address, the amount, and
can trace subsequent token movements. This is identical to every
existing vesting protocol.

When claiming to a **private account**, the vesting program
credits the beneficiary's private account directly. Observers see
that a claim occurred (amount and timestamp) but cannot determine
which private account holds the tokens or trace subsequent
movements.

#### What is public (observable on-chain)

- All vesting schedule parameters: token, total amount, schedule
  type, cliff date, end date, beneficiary address.
- All claim events: amount claimed and timestamp.
- Cancellations and milestone signals.

#### What is private (when claiming to a private account)

- Which private account holds the claimed tokens.
- Subsequent movements of claimed tokens.

#### Privacy limitation

The beneficiary address is permanently and publicly associated
with the vesting schedule from the moment it is created. Every
claim from the schedule is observably linked to that address.
The privacy guarantee covers post-claim token movements only:
the destination private account and all subsequent activity are
untraceable, but the beneficiary's association with the vesting
position cannot be hidden. For full beneficiary anonymity,
commitment-based beneficiary registration (depending on LP-0003)
would be required; this is deferred to a future RFP.

### Soft Requirements

- The creator can nominate a separate cancellation authority
  distinct from the creator wallet (useful when vesting is managed
  by a multisig or DAO).
- The creator can nominate a separate milestone authority distinct
  from the creator wallet. Creator-controlled milestone authority
  is appropriate for team vesting, but creates a conflict of
  interest when vesting is used for buyer protection (e.g., creator
  collateral vesting from a token launch per
  [RFP-015](./RFP-015-bonding-curve-launchpad.md) or
  [RFP-016](./RFP-016-lbp-launchpad.md)): the creator could
  approve milestones and drain funds immediately. Delegating
  milestone authority to a governance body or multisig that
  excludes the creator prevents this.

## ⚠ Platform Dependencies

LEZ has similar programming capabilities to Solana but several
primitives required by a vesting protocol are not yet available.
Development is blocked until the dependencies below are resolved.

### Hard dependencies

#### On-chain clock / timestamp

LEZ does not yet have on-chain block time. Vesting schedules are
fundamentally time-based: the cliff date, linear accrual rate, and
total vested amount at any moment are all functions of elapsed time.
Without a reliable on-chain timestamp, the program cannot determine
how much of a schedule has vested, making cliff + linear and fully
linear schedule types impossible to implement correctly.
Milestone-based vesting does not require an on-chain timestamp and
is unaffected by this dependency, but it represents only a subset of
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

### Soft dependencies

Desirable but not required to begin development.

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
- Familiarity with privacy interaction patterns (public and private
  accounts, see RFP-008)

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
- [Appendix: Token Vesting Ecosystem](../appendix/token-vesting-ecosystem.md)
  (survey of existing vesting protocols, scale data, and fee structures)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
