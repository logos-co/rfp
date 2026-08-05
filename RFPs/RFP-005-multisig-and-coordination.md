---
id: RFP-005
title: Multisig and Coordination
tier: XL
status: open
dependencies:
  - id: LP-0002
    reason: LP-0002 is an open prize for a private M-of-N multisig primitive on LEZ; it is adjacent work, not a delivered capability, and no end-to-end multi-party authorization flow exists in the LEZ codebase today. Its anonymous-approval design differs from this RFP's attributed approvals (R.1), so it is not a drop-in foundation. What this RFP actually requires is private-account execution, which is available.
category: Applications & Integrations
---

# RFP-005 — Multisig and Coordination

## 🧭 Overview

Build a production-ready M-of-N multisig program on the Logos Execution Zone
(LEZ), together with an in-band coordination channel so signers can propose,
deliberate, collect approvals, and reach quorum without leaving the application.
A multisig is the execution layer for shared custody, treasuries, and DAOs. The
program is designed and implemented from scratch for LEZ, taking as its baseline
the properties LEZ makes uniquely possible: multisig state that is private by
default, and a coordination channel that is encrypted and metadata-resistant.

Multisig is the single most widely used custody primitive in the ecosystem, and
the value in multisig custody today is immense. Safe (formerly Gnosis Safe), the
dominant standard on Ethereum, is deployed across 86 networks and self-reports
over US$60B in assets secured; Squads, the Solana analogue, self-reports over
US$15B (protocol-reported figures, 2026-08-03). Both are fully transparent by
construction: members, threshold, every approval, and every action are public
on-chain state, because the chain must read the quorum in order to enforce it.
That transparency has been an attack surface: the Bybit (February 2025,
~US$1.5B) and WazirX (July 2024, ~US$235M) incidents both targeted fully visible
Safe configurations at the signing layer.

The team building this should have deep experience in multisig or threshold
custody design, Rust program development for a RISC-V or zkVM target, and
applied cryptography.

## 🔥 Why This Matters

Shared custody is a precondition for organisations to operate on Logos. Without
a production multisig, there is no treasury, no DAO execution layer, and no
shared control of protocol admin authorities.

LEZ also makes it possible to close a gap no sovereign multisig has closed.
Every existing implementation that keeps multisig structure private (FROST,
MuSig2 n-of-n) does so by moving the quorum off-chain into a cryptographic
session, at the cost of hardware immaturity and, for MuSig2, an n-of-n-only
limitation. Every implementation that keeps the quorum on-chain is transparent.
On LEZ, the same multisig program can run over private accounts, so the chain
records only a commitment to the post-state and a validity proof, without giving
up k-of-n. Under the all-members key-custody model it achieves this without a
trusted coordinator; the alternative custody model trades that property away for
stricter role separation, and the choice is left to the proposer. This RFP is
the vehicle for delivering that.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

01. Implement an M-of-N multisig program on LEZ. A multisig is created with N
    members and a threshold M; an action requires at least M approvals before it
    can execute. Members, threshold, and the action set are configurable at
    creation.
02. Support the full proposal lifecycle. Proposals and approvals are coordinated
    off-chain through the per-multisig coordination room (requirement F.9): a
    member publishes a proposal to the room; members approve or reject
    asynchronously; once M approvals are collected the action can be executed,
    and the program verifies the collected approvals at execution. Proposals
    carry an expiry after which the program rejects execution.
03. Execute approved actions on any arbitrary program deployed in the given LEZ.
04. Provide a registry for proposal code and target programs so any client can
    confirm what instructions a proposal's bytes represent and what program they
    invoke, without relying on a single trusted source. Note that Usability
    requirement U.6 (decode the action before signing) can only be as complete
    as this registry's coverage; proposers must state what a client displays
    when a target program is not registered.
05. Support configuration changes (add or remove a member, change the threshold)
    through the same M-of-N approval flow, so structural changes cannot bypass
    the quorum.
06. Support role separation among members so that the ability to propose, to
    approve, and to execute can be assigned independently. At minimum, a
    proposing key need not be an approving key. Documentation must state which
    guarantee the chosen vault custody model actually delivers for each role.
07. Support an optional per-multisig time lock: a configurable delay between an
    action reaching quorum and becoming executable, enforced by the program. A
    time lock of zero means immediate execution.
08. Support an optional spending-limit policy: a member or sub-quorum may
    execute transfers up to a configured limit without the full M-of-N approval.
09. Provision an end-to-end-encrypted coordination room per multisig using the
    Logos chat module, scoped to the multisig's members. The room carries both
    human deliberation and machine coordination: proposals are published to the
    room, member approvals are collected through it, and the resulting approval
    package is presented to the program at execution. See the Coordination
    Architecture section.
10. Run the multisig private by default: the program runs over LEZ private
    accounts so that the multisig data items listed in the Privacy Architecture
    section are not published on-chain. Support the auditability and
    transparency options defined there: an operator-selectable fully public
    posture, and selective disclosure of a private multisig's state to a chosen
    audience.

#### Usability

1. Provide an SDK that can be used to build Logos modules for interacting with
   the multisig (create, propose, approve, reject, execute, manage members and
   policies).
2. Provide a Logos mini-app QML GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. The mini-app must
   surface the per-multisig coordination room alongside the proposal list.
3. Provide a CLI that uses the Logos core headless framework and covers core
   functionality of the program (create, propose, approve, reject, execute, and
   configuration changes). The CLI may have fewer features than the GUI mini-app
   but must support all essential operations.
4. Provide an IDL for the LEZ program, using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Documentation must clearly explain, for each action, what information is
   public and what is private under the multisig's configured posture (which of
   the data items enumerated in the Privacy Architecture section are published
   on-chain and which are not), so a multisig operator understands exactly what
   an observer can see, including what becomes visible to an audience under each
   disclosure mechanism.
6. Before a member signs an approval, the mini-app must display the exact action
   that approval authorises (target program, decoded instruction, amounts), so
   the member verifies what they are signing rather than a UI-rendered summary.
   This addresses the signing-layer attack surface behind the Bybit and WazirX
   losses, in which signers authorised what a compromised UI showed them rather
   than what was actually executed. It complements the primary mitigation, which
   is the Logos module model itself: the UI is installed and verified once
   rather than fetched from a remote server on every use.
7. Failed or rejected proposals and executions must return clear, actionable
   error messages.

#### Reliability

1. The program must verify, at execution, that the presented approvals are M
   distinct valid approvals from current members on exactly the action being
   executed; no approval is double-counted or replayed across proposals, and an
   action cannot execute with fewer than M valid approvals.
2. A configuration change (member set or threshold) must invalidate approval
   sets gathered under the old configuration: an approval collected before the
   change must not count toward quorum after it, unless the action had already
   reached quorum and entered its time-lock delay or been submitted for
   execution. The proposal must define precisely which of those points is the
   cutoff, and what happens to a pending action when a member is removed during
   a time-lock delay.

#### Performance

1. Compute unit usage, transaction size, and client-side proving time for each
   on-chain operation must be documented and benchmarked against LEZ devnet
   limits. All benchmarks must be produced with **real proving**. Development
   mode produces stub receipts orders of magnitude smaller and faster than real
   ones, and figures gathered that way are meaningless for capacity planning;
   benchmarks submitted from development mode will not be accepted.
2. The cost of verifying approvals at execution must be benchmarked and reported
   as a function of M, since it is on the critical path for proof size and cost.
   Report the largest M that remains viable within block limits. **M of at least
   5 must remain viable** for the deliverable to be considered complete; if the
   benchmark shows otherwise, the finding itself is a reportable result and
   triggers a scope discussion rather than silent delivery of a lower ceiling.

#### Supportability

1. The multisig program is deployed and tested on LEZ devnet/testnet, and is
   compatible with Logos testnet 0.3 and 0.4.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, and Reliability has at
   least one corresponding test. At minimum this includes: an action cannot
   execute below threshold; a configuration change respects requirement R.2; and
   a vault cannot be drained before it is fully initialised. Performance
   requirements are satisfied by reported measurements rather than pass/fail
   tests, but the benchmark harness must be committed and reproducible.
5. A README documents end-to-end usage: deployment steps, program addresses, and
   step-by-step instructions for creating a multisig, proposing, approving, and
   executing via CLI and front-end, including how the coordination room is
   provisioned.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey.
7. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator journey.
8. Provide Figma designs or equivalent for the mini-app GUI, including the
   proposal list and the coordination room.
9. Publish resulting modules in the
   [Logos modules catalog](https://github.com/logos-co/logos-modules-release-base).

### Soft Requirements

If possible.

#### Functionality

1. Support weighted approvals, where members carry numeric weights and the
   threshold is a minimum cumulative weight.
2. **Proof of holding.** Enable a multisig to demonstrate that its vault holds
   at least a stated amount, verified against the on-chain commitment, without
   revealing the exact balance. This is a soft requirement because it is not
   expressible with today's primitives: the on-chain artefact is a hash
   commitment over the whole account, so proving an inequality against it
   requires a second, purpose-built zero-knowledge circuit that does not exist
   and whose verifying key would need to be established and trusted. A proposer
   may scope this as a research deliverable with its own budget, or document a
   design for later implementation. A proposal that omits it entirely is not
   penalised.

### Coordination Architecture

Every multisig provisions one end-to-end-encrypted room using the Logos chat
module, scoped to its members. The room is the single channel for both kinds of
coordination traffic:

- **Human deliberation**: the discussion among signers about whether to approve.
- **Machine coordination**: proposals are published to the room, and members'
  approvals (signatures over the proposal) are collected through it. Once M
  approvals are gathered, the approval package is submitted for execution; the
  program verifies the collected approvals at execution time.

No sovereign multisig in production today offers an encrypted,
metadata-resistant coordination channel: coordination is either public on-chain
state (Squads), a relay that sees the metadata (Safe), or a user-supplied
external channel (Bitcoin). The Logos chat module closes that gap. Carrying
approvals through the room removes per-approval on-chain writes and keeps
pending-proposal metadata off-chain even for a public-posture multisig. An
on-chain proposal record is not an audit-trail advantage over this model: both
models yield a verifiable record of who authorised an action at execution.

Room membership and program membership are separate state and can diverge.
Removing a member through the M-of-N flow (F.5) changes the program's member
set, but does not by itself evict that member from the coordination room, and a
stale client may keep showing them as a participant. Proposals must state how
room membership is reconciled with program membership on every configuration
change, and what a removed member can still observe in the room until that
reconciliation completes.

### Privacy Architecture

A multisig involves at least ten distinct data and metadata items: the existence
of the multisig, the member set, the threshold, per-signer approval attribution,
pending-proposal metadata, the action payload, the vault balance, execution
linkage, coordination content, and the co-signing social graph. On LEZ, each of
these can independently be public or private, because the Logos Execution
Environment runs the same program over public accounts (visible on-chain) or
private accounts (only a post-state commitment and validity proof on-chain).

**Posture: private by default.** Under this RFP the multisig runs over private
accounts by default, so none of the ten items is published in the clear;
coordination content is always private (the E2EE room), and the co-signing
social graph is not readable from chain state. Where identity is concerned, the
property in question is unlinkability between a user's account and its role in a
multisig, not concealment of the user.

**Auditability and transparency options.** Privacy is not the opposite of
oversight, and different organisations need different audiences able to inspect
the multisig. A corporate or organisational treasury typically needs a narrow
audience (auditors, a board) able to inspect it. A DAO treasury typically needs
a wider one: members joining a DAO may reasonably require evidence that the
treasury is secured as its key holders claim, on an ongoing basis rather than
once at setup. The program must support:

1. **Public posture (operator-selectable).** At creation the operator may deploy
   the multisig fully public instead of private, for treasuries that want anyone
   to be able to inspect configuration, holdings, and activity at all times.
2. **Selective disclosure to a defined audience.** The program must enable a
   private multisig to disclose its state (configuration, holdings, activity) to
   a chosen audience without making that information public and without granting
   spending power to that audience. The audience may be narrow (a named auditor)
   or wide (all members of a DAO), and the mechanism must support both. The
   implementer should study and propose the mechanism that best balances
   auditability, security, and usability, and must state which granularity it
   delivers and what an audience unavoidably learns.
3. **Ongoing assurance.** The program must enable a multisig to demonstrate its
   holdings and configuration to its audience repeatedly over time, so that a
   party joining later can obtain current evidence rather than relying on a
   claim made at setup.

Documentation (Usability requirement U.5) must state the resulting
public/private split explicitly for the configured posture, including who can
see what under each disclosure mechanism.

## ⚠ Platform Dependencies

This RFP is open for proposals. Proposers may begin design and development work,
but a working on-chain deployment depends on the platform components below.
Proposers should confirm the current state of each against the Resources section
before relying on it.

- **LEZ private accounts.** The private-by-default posture requires running the
  multisig over LEZ private accounts. Note that no end-to-end multi-party
  authorisation flow exists on LEZ today: the underlying primitives are
  available, but this RFP commissions the first such implementation.

- **Shared private accounts.** LEZ provides group-owned shared private accounts
  derived from a single Group Master Secret, documented in the Journey linked
  under Resources. Proposers should study this feature and state how, and
  whether, they use it. Note that every holder of the group secret derives full
  spending authority over the shared account, so it distributes custody rather
  than dividing it; understanding its properties is a prerequisite to designing
  the vault.

- **Logos chat module.** The per-multisig coordination room (F.9) must be built
  on the Logos chat module. This is mandatory, not a suggested option.

- **Logos testnet compatibility.** The delivered implementation must be
  compatible with Logos testnet 0.3 and 0.4.

### Risks

#### Approval verification cost

The program must verify collected member approvals inside its own execution.
There is no precedent on LEZ to size this against, and the cost scales with M,
so it is the largest unpriced item in this RFP. Proposers should establish this
cost early, before the design is committed. Performance requirement P.2 makes
the benchmark a deliverable.

#### Private-transaction throughput and proving cost

Private-transaction throughput per block is limited, and proof generation is
paid client-side and measured in minutes rather than seconds. This shapes the
product: the mini-app and CLI must treat execution as a long-running background
operation with visible progress, not a request-response interaction. Carrying
approvals through the coordination room keeps the on-chain footprint small, but
proposers must measure and report the real figures under Performance requirement
P.1.

Benchmarks must be produced with real proving. Development mode skips proof
generation and yields figures that are orders of magnitude optimistic, which is
misleading for capacity planning.

#### Signing-layer trust

The Bybit and WazirX losses were not smart-contract failures; they exploited the
gap between what a signer saw in a UI and what they actually authorised. The
primary mitigation is the Logos module model: the UI is installed and verified
once, not downloaded from a remote server on every use. Usability requirement
U.6 (show the exact decoded action before signing) is a secondary mitigation and
must not be treated as optional polish.

## 👤 Recommended Team Profile

Team experienced with:

- Multisig, threshold custody, or account-abstraction design
- Rust program development for a RISC-V or zkVM target (Risc0 experience a plus)
- Applied cryptography and zero-knowledge proof systems
- Secure signing UX and hardware-wallet integration
- Front-end development for custody or wallet applications

## ⏱ Timeline Expectations

Estimated duration: **6 months** (fresh implementation of the M-of-N program
with its private-by-default execution path, the coordination room with in-room
approval collection, and the SDK, CLI, and mini-app). The vault architecture and
privacy posture are settled; vault key custody and the approval-verification
scheme are left to the proposer (see Decisions for Review).

This estimate assumes a team already productive on LEZ. Proposers new to the
platform should account for ramp-up separately and say so.

**A phased proposal is welcome.** The in-guest approval verifier is on the
critical path, has no precedent to size against, and its cost determines the
largest workable M. Proposers may structure the work so that an initial phase
establishes the approval-verification benchmark (P.2), the account layout
against the private-account ceiling, and the time-lock design, with the scope
and cost of the remainder fixed once those are known. A proposal that names this
uncertainty and structures around it will be viewed more favourably than one
that prices it silently.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [logos-co/lez-multisig](https://github.com/logos-co/lez-multisig): a public
  multisig proof-of-concept sample app on LEZ; prior art only — this RFP
  commissions a fresh design and implementation. Note that its architecture is
  incompatible with private accounts: it requires member accounts to be fresh
  zero-nonce keypairs claimed by the multisig program, which private accounts
  cannot satisfy because they are owned by the privacy protocol and increment
  the nonce on every use. Treat it as a reference for the public path only.
- **LP-0002, Private M-of-N Multisig** (open λ prize): commissions a private
  M-of-N primitive for LEZ using anonymous threshold proofs, where the verifier
  confirms a threshold was met without recording which members approved. It
  overlaps this RFP's ground but is unclaimed, and its anonymity model differs
  from Reliability requirement R.1 here, which requires approvals attributable
  to current members. A proposer should read it for the design space it maps —
  threshold proof schemes, nullifier design, and the LEZ nonce constraint — not
  as a component this RFP builds on.
- [Introduction to the Logos Execution Zone](https://docs.logos.co/lez): the
  public/private account model this RFP relies on
- [Logos Chat Module](https://docs.logos.co/messaging/chat-module/build-logos-module-that-uses-chat-module-api):
  documentation for building modules that use the chat module API
- [Journey: Allow different users to interact with same private account](https://github.com/logos-co/logos-docs/issues/321):
  official Logos journey documenting the shared private account feature

## 🧩 Decisions for Review

Settled and embedded in the requirements above:

1. The privacy posture is **private by default**, with the auditability and
   transparency options listed in Privacy Architecture.
2. **Approvals flow through the E2EE coordination room**, with the program
   verifying the collected approvals at execution (no per-approval on-chain
   writes).
3. The vault is **controlled by the multisig program**, with M-of-N enforced by
   the program's verified execution rather than by how the vault's keys are
   distributed. Member changes are program state changes, not key migrations.

Group-shared accounts derived from a Group Master Secret are **not** the vault
mechanism: every holder of the group secret gets full spending authority, so a
vault built on one would be advisory rather than enforcing. The feature remains
useful for shared **viewing** of vault activity and for keying the coordination
room, and proposers may use it for those purposes.

### Open for the proposer to decide

**Vault key custody.** A spend must ultimately be constructed by some party
holding the vault's spending key. Both models below are viable; the proposer
must choose one, justify it, and document the resulting threat model:

- **All members hold the vault spending key.** Any member can construct a spend
  transaction, but the program rejects it below quorum, so funds are safe once
  the vault is initialised. No liveness dependency, no single point of key loss,
  and no trusted coordinator. The cost is that "execute" is not a separable role
  under F.6, since every member holds what is needed to submit.
- **A designated operator or relayer holds the vault spending key.** Members
  hold approval keys only and cannot construct a spend transaction at all, which
  makes F.6 role separation fully meaningful. The cost is a trusted coordinator:
  a liveness dependency, a single point of key loss, and a censorship vector.
  Choosing this model forfeits the no-trusted-coordinator property claimed in
  Why This Matters, and the documentation must say so plainly.

Neither model provides cryptographic k-of-N at the key layer. That requires
threshold cryptography, noted below as a future extension.

**Approval verification scheme.** How member approvals are verified inside the
program is the proposer's choice, subject to the benchmark required by
Performance requirement P.2 and to meeting Reliability requirement R.1.

### Future extension (not a deliverable)

**Threshold cryptography over the vault keys** (for example a FROST-style
scheme) would give cryptographic M-of-N at the key layer, removing the need for
any single party to hold a complete spending key. Tooling is immature and
threshold-signature implementation is out of scope for this RFP; a proposer may
document it as a migration path.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
