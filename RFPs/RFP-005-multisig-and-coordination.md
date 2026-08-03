---
id: RFP-005
title: Multisig and Coordination
tier: L
funding: $XXXXX
status: open
dependencies:
  - id: LP-0002
    reason: LP-0002 demonstrates an M-of-N multisig running over LEE private
      accounts; this RFP's private-by-default posture relies on that execution
      model being production-available on LEZ.
category: Applications & Integrations
---

# RFP-005 — Multisig and Coordination

## 🧭 Overview

Build a production-ready M-of-N multisig program on the Logos Execution Zone
(LEZ), together with an in-band coordination channel so signers can propose,
deliberate, collect approvals, and reach quorum without leaving the
application. A multisig is the execution layer for shared custody, treasuries,
and DAOs: a Private DAO Lambda Prize will be published and expected to use the
this RFP's implementation. The program is designed and implemented from scratch for LEZ,
taking as its baseline the properties LEZ makes uniquely possible: multisig
state that is private by default, and a coordination channel that is encrypted
and metadata-resistant.

Multisig is the single most widely used custody primitive in the ecosystem, and
the value in multisig custody today is immense. Safe (formerly Gnosis Safe),
the dominant standard on Ethereum, is deployed across 86 networks and
self-reports over US$60B in assets secured; Squads, the Solana analogue,
self-reports over US$15B (protocol-reported figures, 2026-08-03). Both are
fully transparent by construction: members, threshold, every approval, and
every action are public on-chain state, because the chain must read the quorum
in order to enforce it. That transparency has been an attack surface: the
Bybit (February 2025, ~US$1.5B) and WazirX (July 2024, ~US$235M) incidents
both targeted fully visible Safe configurations at the signing layer.

The team building this should have deep experience in multisig or threshold
custody design, Rust program development for a RISC-V or zkVM target, and
applied cryptography.

## 🔥 Why This Matters

Shared custody is a precondition for organisations to operate on Logos. Without
a production multisig, there is no treasury, no DAO execution layer, and no
shared control of protocol admin authorities. The launch-day Private Multisig
app and the Private DAO are blocked on it.

LEZ also makes it possible to close a gap no sovereign multisig has closed.
Every existing implementation that keeps multisig structure private (FROST,
MuSig2 n-of-n) does so by moving the quorum off-chain into a cryptographic
session, at the cost of hardware immaturity and, for MuSig2, an n-of-n-only
limitation. Every implementation that keeps the quorum on-chain is transparent.
On LEZ, the same multisig program can run over private accounts, so the chain
records only a commitment to the post-state and a validity proof, without a
trusted coordinator and without giving up k-of-n. This RFP is the vehicle for
delivering that.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement an M-of-N multisig program on LEZ. A multisig is created with N
   members and a threshold M; an action requires at least M approvals before it
   can execute. Members, threshold, and the action set are configurable at
   creation.
2. Support the full proposal lifecycle. Proposals and approvals are coordinated
   off-chain through the per-multisig coordination room (requirement F.8): a
   member publishes a proposal to the room; members approve or reject
   asynchronously; once M approvals are collected the action can be executed,
   and the program verifies the collected approvals at execution. Proposals
   carry an expiry after which the program rejects execution.
3. Execute approved actions on any arbitrary program deployed in the given LEZ:
   a proposal designates a target program and an instruction, and on execution
   the multisig program invokes the target on the multisig's authority. The
   multisig never modifies another program's state directly. The design must
   not special-case any particular target program.
4. Support configuration changes (add or remove a member, change the threshold)
   through the same M-of-N approval flow, so structural changes cannot bypass
   the quorum.
5. Support role separation among members so that the ability to propose, to
   approve, and to execute can be assigned independently. At minimum, a
   proposing key need not be an approving key.
6. Support an optional per-multisig time lock: a configurable delay between an
   action reaching quorum and becoming executable, enforced by the program. A
   time lock of zero means immediate execution.
7. Support an optional spending-limit policy: a member (or sub-quorum) may
   execute transfers up to a configured limit without the full M-of-N approval.
8. Provision an end-to-end-encrypted coordination room per multisig using the
   Logos chat module, scoped to the multisig's members. The room carries both
   human deliberation and machine coordination: proposals are published to the
   room, member approvals are collected through it, and the resulting approval
   package is presented to the program at execution. See the Coordination
   Architecture section.
9. Run the multisig private by default: the program runs over LEE private
   accounts so that the multisig data items listed in the Privacy Architecture
   section are not published on-chain. Support the auditability and
   transparency options defined there: an operator-selectable fully public
   posture, and selective disclosure of a private multisig's state to a chosen
   audience via view keys.

#### Usability

1. Provide an SDK that can be used to build Logos modules for interacting with
   the multisig (create, propose, approve, reject, execute, manage members and
   policies).
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. The mini-app must
   surface the per-multisig coordination room alongside the proposal list.
3. Provide a CLI that covers core functionality of the program (create, propose,
   approve, reject, execute, and configuration changes). The CLI may have fewer
   features than the GUI mini-app but must support all essential operations.
4. Provide an IDL for the LEZ program, preferably using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Documentation must clearly explain, for each action, what information is
   public and what is private under the multisig's configured posture (which of
   the data items enumerated in the Privacy Architecture section are published
   on-chain and which are not), so a multisig operator understands exactly what
   an observer can see, including what becomes visible to an audience once a
   view key is shared with it.
6. Before a member signs an approval, the mini-app must display the exact action
   that approval authorises (target program, decoded instruction, amounts), so
   the member verifies what they are signing rather than a UI-rendered summary.
   This is the primary mitigation for the signing-layer attack surface behind
   the Bybit and WazirX losses: in both incidents, signers authorised what a
   compromised UI showed them, not what was actually executed.
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
   reached quorum and been scheduled for execution.

#### Performance

1. Each on-chain operation (create, execute, configuration change) completes
   within a single LEZ transaction. Proposal publication and approval
   collection happen in the coordination room, not on-chain.
2. Compute unit usage and transaction size of each on-chain operation must be
   documented and benchmarked against LEZ devnet limits. Because execution runs
   in the private path by default, the per-proof cost and the per-block private
   transaction throughput must be measured and reported.

#### Supportability

1. The multisig program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test, including a test that an
   action cannot execute below threshold and a test that a configuration change
   respects requirement R.2.
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

### Soft Requirements

If possible.

#### Functionality

1. Support batch proposals: a single proposal covering multiple actions that are
   approved once and executed atomically or in sequence.
2. Support weighted approvals, where members carry numeric weights and the
   threshold is a minimum cumulative weight.
3. Support zero-knowledge proof-of-holding: the multisig can prove its vault
   holds at least a stated amount, verified against the on-chain commitment,
   without revealing the exact balance (see the auditability options in Privacy
   Architecture).

### Out of Scope

The following are explicitly excluded from this RFP:

- A FROST or MuSig2 threshold-signature implementation. Structural privacy on
  LEZ comes from running the multisig program over private accounts, not from
  an off-chain threshold-signature session. A threshold scheme layered over
  shared-account keys remains a documented future extension (see the Decision
  for Review), but no threshold-signature implementation is delivered here.
- Recovery of a multisig whose members have lost quorum-many keys. Social or
  time-locked key recovery is a separate concern and is not required here.

### Coordination Architecture

Every multisig provisions one end-to-end-encrypted room using the Logos chat
module, scoped to its members. The room is the single channel for both kinds of
coordination traffic:

- **Human deliberation**: the discussion among signers about whether to
  approve.
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

The room is coordination, not enforcement: quorum is always verified and
enforced by the program at execution, never by the chat channel. Because there
is no live on-chain pending view, the client must reliably gather, retain, and
present room state so signers always see the current proposal set and collected
approvals.

### Privacy Architecture

A multisig involves at least ten distinct data and metadata items: the
existence of the multisig, the member set, the threshold, per-signer approval
attribution, pending-proposal metadata, the action payload, the vault balance,
execution linkage, coordination content, and the co-signing social graph. On
LEZ, each of these can independently be public or private, because the Logos
Execution Environment runs the same program over public accounts (visible
on-chain) or private accounts (only a post-state commitment and validity proof
on-chain).

**Posture: private by default.** Under this RFP the multisig runs over private
accounts by default, so none of the ten items is published in the clear;
coordination content is always private (the E2EE room), and the co-signing
social graph is not readable from chain state. Where identity is concerned, the
property in question is unlinkability between a user's account and its role in
a multisig, not concealment of the user.

**Auditability and transparency options.** Privacy is not the opposite of
oversight, and different organisations need different audiences able to inspect
the multisig: a corporate or org multisig may need a limited auditor group,
while a DAO may need its entire membership to be able to audit. The program
must support:

1. **Public posture (operator-selectable).** At creation the operator may
   deploy the multisig fully public instead of private, for treasuries that
   want anyone to be able to inspect configuration, holdings, and activity at
   all times.
2. **Selective disclosure via view keys.** LEZ private accounts separate the
   spending key from a viewing key that decrypts the account's on-chain state
   without granting spending authority. Distributing the viewing key to a
   defined audience (for example, to DAO members through the coordination room
   or another member-restricted channel) lets that audience inspect the
   multisig's holdings and configuration at any time, without making them
   public and without granting spending power. A member holding the viewing key
   can also demonstrate to a new joiner that the treasury holds what the key
   holders claim, which serves as standing proof of holding. View-key
   distribution is not currently exposed by the LEZ wallet, so the SDK must
   implement it. A shared viewing key cannot be revoked: rotating the audience
   requires migrating the multisig to a fresh account, and the documentation
   must say so.

Documentation (Usability requirement U.5) must state the resulting
public/private split explicitly for the configured posture, including who can
see what once a view key has been shared.

## ⚠ Platform Dependencies

This RFP is open for proposals. Proposers may begin design and development work,
but a working on-chain deployment depends on the primitives below.

### Hard blockers

#### Private-account execution (core LEE feature)

The private-by-default posture requires running the multisig over LEE private
accounts: post-state commitments, nullifiers, and Risc0 validity proofs. These
are core LEE features, demonstrated for an M-of-N multisig under LP-0002;
proposers should confirm the state of private-account support on LEZ devnet
against the Resources below before relying on it, including whether
program-derived private accounts are supported for the multisig's vault — that
property determines the options in the Decision for Review below.

#### Shared private accounts (key separation and group keys)

The auditability options rely on LEE key separation: a private account has a
spending (nullifier) key and a viewing key, and sharing the viewing key yields
a view-only auditor. LEZ also provides group-owned shared private accounts
derived from a single shared group secret, whose semantics matter for the vault
design: every member of the group derives full spending authority (there is no
view-only or threshold share of the group secret), group membership is not
recorded on-chain, and there is no revocation short of migrating funds to a new
account. Proposers should confirm these properties against the LEZ
documentation and codebase before relying on them.

#### Logos chat module

The per-multisig coordination room (Functionality requirement F.8) depends on
the Logos chat module. Proposers should confirm the module's availability and
its SDK surface for creating a member-scoped, end-to-end-encrypted room before
relying on it.

### Resolved dependencies

#### Cross-program execution

Execution of an arbitrary LEZ program (Functionality requirement F.3) relies on
the LEZ runtime's cross-program execution support, which is delivered as part
of the core runtime. Proposers should confirm its mechanism and constraints on
devnet.

### Risks

#### Private-transaction throughput

Running over private accounts, each on-chain operation produces a proof and
consumes a private-transaction slot. LEZ private-transaction throughput per
block is limited (as of 2026-04, one private transaction per block on the
reference deployment). Carrying approvals through the coordination room removes
per-approval on-chain writes, so the on-chain footprint stays small (creation,
executions, configuration changes), but it remains bounded by this throughput.
Performance requirement P.2 requires this to be measured and reported.

#### Signing-layer trust

The Bybit and WazirX losses were not smart-contract failures; they exploited
the gap between what a signer saw in a UI and what they actually authorised.
Usability requirement U.6 (show the exact decoded action before signing) is the
primary mitigation and must not be treated as optional polish.

## 👤 Recommended Team Profile

Team experienced with:

- Multisig, threshold custody, or account-abstraction design
- Rust program development for a RISC-V or zkVM target (Risc0 experience a plus)
- Applied cryptography and zero-knowledge proof systems
- Secure signing UX and hardware-wallet integration
- Front-end development for custody or wallet applications

## ⏱ Timeline Expectations

Estimated duration: **12 weeks** (fresh implementation of the M-of-N program
with its private-by-default execution path, the coordination room with in-room
approval collection, and the SDK, CLI, and mini-app). The privacy posture and
the coordination channel are settled (see below); the remaining open decision
affects the vault design and sits within this estimate.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [logos-co/lez-multisig](https://github.com/logos-co/lez-multisig): a public
  multisig proof-of-concept sample app on LEZ; prior art only — this RFP
  commissions a fresh design and implementation
- [Introduction to the Logos Execution Zone](https://docs.logos.co/lez): the
  public/private account model this RFP relies on

## 🧩 Decisions for Review

Two decisions that shaped earlier drafts are now settled and embedded in the
requirements above: the privacy posture is **private by default**, with the
auditability and transparency options listed in Privacy Architecture; and
**approvals flow through the E2EE coordination room**, with the program
verifying the collected approvals at execution (no per-approval on-chain
writes). One decision remains open.

### Decision — M-of-N enforcement for the private vault

A private multisig needs somewhere to hold value and somewhere to enforce the
quorum, and LEZ's shared-private-account mechanism constrains both. What is
known today: a LEZ private account can be shared through a single group secret
from which every member derives full spending and viewing authority —
effectively N-of-N, not M-of-N; the chain cannot distinguish a group-owned
account from a single-owner one; and there is no revocation short of migrating
funds to a fresh account. A genuine M-of-N must therefore be enforced somewhere
other than the group secret itself.

- **Option A — Program-enforced M-of-N over a program-derived private
  account.** The vault is a private account controlled by the multisig program;
  the M-of-N check runs inside the program's verified execution, and members
  hold no independent spending authority over the vault. The group-secret
  mechanism is not used for spending. Strongest enforcement, and member changes
  are program state changes rather than key migrations; depends on
  program-derived private accounts being supported on LEZ.
- **Option B — Group-shared account as vault, M-of-N enforced in the approval
  layer.** The vault is a group-shared private account and the multisig gates
  which spends are authorised through its approval flow. Simpler account and
  key-distribution story, but every group member cryptographically retains full
  spending authority, so enforcement is procedural rather than cryptographic,
  and excluding a member requires migrating the vault.
- **Option C — Group-shared account as vault, with threshold cryptography over
  the group-derived keys** (for example a FROST-style scheme), giving
  cryptographic M-of-N at the key layer. Strongest key-layer enforcement, but
  immature tooling; threshold-signature implementation is out of scope for this
  RFP, so this would be a documented future extension rather than a
  deliverable.

The proposer should state which baseline their design assumes; the choice will
be settled at contracting time.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
