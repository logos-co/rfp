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

1. The multisig program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, and Reliability has at
   least one corresponding test. At minimum this includes: an action cannot
   execute below threshold; a configuration change respects requirement R.2; and
   a vault cannot be drained through the pre-initialisation window described in
   Platform Dependencies. Performance requirements are satisfied by reported
   measurements rather than pass/fail tests, but the benchmark harness must be
   committed and reproducible.
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
but a working on-chain deployment depends on the primitives below.

### Hard blockers

#### Private-account execution (core LEE feature)

The private-by-default posture requires running the multisig over LEE private
accounts: post-state commitments, nullifiers, and Risc0 validity proofs. These
are core LEE features, and the private-PDA lifecycle this RFP depends on is
covered by integration tests on LEZ `main`, for a single keyholder. Proposers
should confirm the state of private-account support on LEZ devnet against the
Resources below before relying on it.

No end-to-end multi-party authorization flow exists in the LEZ repository today.
The primitives are present and the single-keyholder private-PDA lifecycle is
tested, but nothing demonstrates M-of-N in the program layer. This RFP
commissions the first such implementation. LP-0002 is an open prize covering
adjacent ground, but it is unclaimed and its design differs from this RFP's; see
Resources.

#### Program-derived private PDAs (the enforcement mechanism)

The vault is a private PDA derived under the multisig program's ID. This is what
makes M-of-N enforceable, and the mechanism has two parts, both present and
exercised on LEZ `main`:

1. **Address binding.** A private PDA's account ID commits to the program:
   `AccountId::for_private_pda(program_id, seed, npk, vpk, identifier)`. The
   privacy-preserving circuit refuses any private-PDA pre-state whose npk has
   not been proven to derive that account ID. Three binding paths exist: the
   program's own `Claim::Pda(seed)`, a caller's `pda_seeds`, and an external
   seed supplied directly as a circuit input. The third is what makes funding a
   program-owned private PDA from another program possible, and proposers will
   need it for vault funding.
2. **Ownership latch.** `validate_execution` permits a balance decrease only
   when `account_program_owner == executing_program_id`. Once the vault's
   `program_owner` is set to the multisig program, that latch is one-way:
   ownership cannot be changed or reverted. Only the multisig program's verified
   execution can move funds out.

Enforcement therefore comes from the ownership latch, not from anything about
the key layer. A multisig program that asserts its M-of-N condition makes the
proof unconstructible when the condition fails.

**The vault cannot be claimed by another program.** Because the account ID
commits to the program ID, and the claim path asserts that the account ID
re-derives from the *executing* program's own ID, a hostile program can only
claim accounts within its own namespace. Claiming the multisig's vault would
require a hash preimage collision. This is worth stating explicitly because the
circuit does not enforce authorization claims on private accounts generally: for
the private non-PDA path, `Claim::Authorized` is a deliberate no-op
("unauthorized private claiming is intentionally allowed"). That no-op does not
apply to the private-PDA claim path, which is address-bound.

**What the circuit does not do.** It does not verify that a program's approval
predicate ran and returned true. It proves derivation-from-program plus the
seed/npk binding. The approval logic itself is the program's responsibility, and
the ownership latch is what forces the program to run at all.

**Initialisation is security-critical.** The latch protects the vault only once
the multisig program has claimed it. An account that has been funded but not yet
claimed sits under the default program owner, where the balance-decrease rule
does not yet bind it to the multisig program, and any holder of the vault
spending key can move funds out before the program is ever involved. Vault
creation and first funding must therefore be atomic, or the design must make
pre-initialisation deposits impossible. Proposers must state how they achieve
this, and Reliability testing must cover it.

**Vault key custody is an open design choice.** The circuit has no notion of an
npk that nobody holds: someone must hold the vault's nullifier secret key (nsk)
to construct the spend proof. Two models are viable and the proposer must choose
one and justify it (see Decisions for Review).

#### Shared private accounts and group keys (not a multisig primitive)

LEZ supports group-owned shared private accounts derived from a single 32-byte
Group Master Secret (GMS): every member independently derives the same account
keys (NSK/VSK/NPK/VPK). New members are admitted by sealing the GMS to their
public key and having them unseal it.

**The GMS confers full spending authority and cannot be restricted to viewing.**
Because the derivation hands every holder the full nsk, and holding the nsk is
what permits a spend, distributing the GMS to N members distributes N copies of
full spending authority. It is not a multisig primitive and must not be used as
one. Where a view-only auditor is wanted, share the account's viewing key
instead — that is the mechanism that actually separates viewing from spending.

Specifically, a **regular** (non-PDA) GMS-derived shared account carries no
program binding at all: its account ID is derived without a program ID, and its
`program_owner` ends up as the standard funding program. Any single GMS holder
can spend it directly without the multisig program ever being invoked. A
multisig built on that account shape would be advisory, not enforcing.

Properties proposers should account for:

- The key layer is effectively **1-of-N**: any GMS holder derives full spending
  and viewing authority. There is no view-only or threshold share of the GMS.
- The GMS is a **root** secret. A holder derives keys for every account the
  group creates under it, including accounts created after they joined, and can
  re-seal the GMS to an arbitrary third party without the other members'
  consent.
- Group membership is **not recorded anywhere** — not on-chain, and not in any
  registry or distribution service. `invite` prints a sealed blob and `join`
  accepts one; conveying it is entirely the operator's problem, and each
  member's roster is independent local state that can silently diverge.
- There is **no member revocation**. Removing a member deletes only the caller's
  own local copy; the removed member's GMS still derives working spending keys.
  In-place GMS rotation is structurally impossible, because the account ID
  commits to the derived npk and vpk — a new GMS is a different account.
  Migration to a fresh GMS is the only path, it is not automated, and during the
  sweep the removed member holds equal spending authority over the funds being
  moved. They also retain permanent viewing access to the old account's history.

The auditability options rely on LEE key separation: a private account has a
spending (nullifier) key and a viewing key, and sharing the viewing key yields a
view-only auditor. Note that the GMS itself cannot express this separation — any
GMS holder gets both.

Proposers should confirm these properties against the LEZ codebase before
relying on them.

#### Private-account count is part of the anonymity set

A privacy-preserving transaction pads its private inputs to a fixed count, on
the order of seven accounts. Beyond that ceiling the padding saturates and the
number of private accounts a transaction touches stops being hidden, which
weakens the privacy posture this RFP is built on.

This is a hard constraint on account layout, and it binds sooner than proposers
expect. A vault, a policy account, a proposal account, a spending-limit
accumulator, and a transfer recipient already approach the ceiling before any
per-member state exists. Designs that allocate an account per member (see F.8)
will exceed it.

Proposers must state the maximum number of private accounts any single operation
touches, and design the account layout to stay within the ceiling.

#### Time: no clock is readable from the private path

Three requirements depend on time: proposal expiry (F.2), the time lock (F.7),
and spending limits if the limit is per-period (F.8). LEZ offers no clock that a
program can read from the private path.

Clock accounts exist, but they are **public** accounts. Taking one as a
pre-state in a privacy-preserving transaction puts a public account in the
transaction, which defeats the private posture and is a strong deanonymisation
signal, since every private transaction reading the clock reads the same
account.

The mechanism that does work is the **timestamp validity window**: a program
declares that its output is valid only within a stated time range, and the state
machine rejects the transaction outside that range. This constrains the
transaction rather than letting the program read the current time. It is
sufficient for a time lock, because an unlock time known at proof-construction
time can be expressed as a window that opens at the unlock point, and the chain
rejects anything proved for an earlier window.

Proposers must design F.2, F.7, and F.8 around validity windows and durable
program state. A team that goes looking for a clock API will not find a usable
one.

#### Logos chat module

The per-multisig coordination room (Functionality requirement F.9) depends on
the Logos chat module. The module is **not part of the LEZ repository**, so its
availability, SDK surface, and support for machine-readable payloads cannot be
confirmed from the LEZ codebase. Proposers must confirm all of this against the
module's own documentation and maintainers before relying on it. Alongside the
in-guest verifier, this is the largest external dependency risk in this RFP.

### Risks

#### In-guest signature verification is unbuilt

Requirement F.2 has the program verify collected member approvals at execution.
**No primitive for this exists today.** Guest programs depend on `lee_core` and
`risc0-zkvm` only, and `lee_core` carries no elliptic-curve library; there is no
signature-verification code reachable from guest code anywhere in the LEZ tree.
The BIP-340/secp256k1 verifier that does exist is host-side, outside the zkVM,
and verifies the transaction witness set rather than program-level semantics.
The only cryptography available in-guest is SHA-256, plus ML-KEM and ChaCha20
for the encryption path.

This is proposer scope rather than a platform blocker: a `no_std` verifier must
be vendored into the guest, paying the RISC Zero cycle cost for M verifications
inside the proof. That is feasible in principle, but there is no precedent in
the repository to size it against, and the cost scales with M. It is the largest
unpriced item in this RFP, and Performance requirement P.2 makes the benchmark a
deliverable.

Because the cost is unknown, proposers are encouraged to establish it early and
to consider whether in-guest signature verification is required at all. An
approval scheme built on the SHA-256 already available in-guest may satisfy
Reliability requirement R.1 far more cheaply; see Decisions for Review.

This gap is not specific to multisig. Any program needing to verify
authorisation in its own execution will meet it, so a verifier built here has
value beyond this RFP. Proposers should design it as a reusable component rather
than a private detail of the multisig program.

#### Private-transaction throughput

Running over private accounts, each on-chain operation produces a Risc0 receipt
and consumes block capacity. The reference sequencer configuration bounds a
block at 20 transactions and 1 MiB, with no separate limit for private
transactions. The block-size bound is the one that binds: a real-proving
privacy-preserving transaction measures roughly 220 KiB on the wire, so
**approximately four private transactions fit in a block** and the
20-transaction count is never reached. At the reference 10-second block cadence
that is on the order of **0.4 private transactions per second for the entire
zone**, shared with every other application running on it.

Proving cost is the more severe constraint, and it is paid client-side at
submission rather than by the sequencer. Published LEZ benchmarks put a single
privacy-preserving transaction at roughly **two minutes of proving on a
commodity laptop CPU**, with each chained call adding roughly a further minute.
A multisig execution that chains into a target program should therefore be
expected to cost several minutes of local proving. This shapes the product: the
mini-app and CLI must treat execution as a long-running background operation
with visible progress, not a request-response interaction.

Carrying approvals through the coordination room removes per-approval on-chain
writes, so the on-chain footprint stays small (creation, executions,
configuration changes). Performance requirement P.1 requires these figures to be
re-measured on the target deployment and reported, since they will have moved.

Note that proving cost scales with power-of-two-bucketed total cycles, not raw
cycle count: reducing cycles lowers cost only when it crosses a bucket boundary.
Benchmarks under Performance requirement P.2 must therefore be reported as a
step function rather than a linear fit.

#### Development mode hides the real cost

LEZ supports a development mode that skips real proving. It is the right way to
run most tests, but it produces stub receipts orders of magnitude smaller and
faster than real ones. A team that develops and benchmarks exclusively in
development mode will carry figures roughly two orders of magnitude optimistic
and discover the true cost only at integration, when the account layout and
approval scheme are already fixed.

Proposers should establish real-proving measurements early, before the design is
committed, and Performance requirement P.1 makes real-proving benchmarks a
condition of acceptance.

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
  official Logos journey documenting the GMS-based shared private account
  feature. Note that its stated release status is out of date: the feature is
  bundled in the released `v0.2.0` tag (2026-06-30), not only on `main`. Take
  care with the tag list — `v0.3.0` predates this work and is not a newer
  release.

Account derivation constants are versioned and have changed between LEZ
releases; public and private PDA derivations do not currently share a version
prefix. Any change to these constants changes every derived account address.
Proposers should pin the derivation they build against with their own tests
rather than assuming stability across releases.

## 🧩 Decisions for Review

Settled and embedded in the requirements above:

1. The privacy posture is **private by default**, with the auditability and
   transparency options listed in Privacy Architecture.
2. **Approvals flow through the E2EE coordination room**, with the program
   verifying the collected approvals at execution (no per-approval on-chain
   writes).
3. The vault is a **private PDA derived under the multisig program's ID**, with
   M-of-N enforced by the program's verified execution, backed by the
   `program_owner` ownership latch described in Platform Dependencies. Member
   changes are program state changes, not key migrations.

Group-shared accounts derived from a Group Master Secret are **not** the vault
mechanism. The GMS distributes full spending authority to every holder, so a
vault built on it would be advisory rather than enforcing. The GMS remains
useful for shared **viewing** of vault activity and for keying the coordination
room, and proposers may use it for those purposes.

### Open for the proposer to decide

**Vault key custody.** Someone must hold the vault's nsk to construct a spend
proof. Neither model below is unambiguously better, and the choice determines
which other requirements can be honoured in full. The proposer must choose one,
justify it, and document the resulting threat model:

- **All members hold the vault nsk.** Any member can construct a spend
  transaction, but the program rejects it below quorum, so funds are safe once
  the vault is initialised. No liveness dependency, no single point of key loss,
  and no trusted coordinator. The costs: "execute" is not a separable role under
  F.6, because every member holds the key needed to submit; members can grief
  each other by racing spends and burning proving effort; and the
  pre-initialisation window described in Platform Dependencies is a real
  exposure that the design must close.
- **A designated operator or relayer holds the vault nsk.** Members hold
  approval keys only and cannot construct a spend transaction at all, which
  makes F.6 role separation fully meaningful and removes the griefing vector.
  The cost is a trusted coordinator: a liveness dependency, a single point of
  key loss, and a censorship vector. Choosing this model forfeits the
  no-trusted-coordinator property claimed in Why This Matters, and the
  documentation must say so plainly rather than implying it still holds.

Neither model provides cryptographic k-of-N at the key layer. That requires
threshold cryptography, noted below as a future extension.

**Approval verification scheme.** Which signature scheme is verified in-guest,
and which `no_std` implementation is vendored, is the proposer's choice, subject
to the benchmark required by Performance requirement P.2. Proposers should also
consider whether in-guest signature verification is needed at all: committing to
an approval set by hash in the program's account data, using the SHA-256 already
available in-guest, may satisfy Reliability requirement R.1 at a fraction of the
proving cost. A proposal that argues for this and shows it meets R.1 is welcome.

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
