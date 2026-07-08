---
id: RFP-005
title: Multisig and Coordination
tier: L
funding: $XXXXX
status: open
dependencies:
  - id: LP-0002
    reason: Private M-of-N multisig sample-app proof; establishes the private-account multisig design this RFP hardens to production.
  - id: SA-public-multisig
    reason: The public multisig sample app (logos-co/lez-multisig) is the transparent baseline whose proposal→approve→execute→ChainedCall design this RFP builds on.
  - id: RD-chat
    reason: The Logos chat module provides the end-to-end-encrypted, metadata-resistant coordination channel required per multisig for signer deliberation and, optionally, signing-material transport.
category: Applications & Integrations
---

# RFP-005 — Multisig and Coordination

## 🧭 Overview

Build a production-ready M-of-N multisig program on the Logos Execution Zone
(LEZ), together with an in-band coordination channel so signers can deliberate
and reach quorum without leaving the application. A multisig is the execution
layer for shared custody, treasuries, and DAOs: the Logos launch-day Private
Multisig app and the Private DAO both depend on it. A public multisig sample app
already exists on LEZ (`logos-co/lez-multisig`, inspired by Squads Protocol v4):
proposals live on-chain, signers approve asynchronously, and execution delegates
to any LEZ program via `ChainedCall`. This RFP hardens that prototype into a
production program and adds the properties LEZ makes uniquely possible: multisig
state that can be kept private, and a coordination channel that is encrypted and
metadata-resistant.

Multisig is the single most widely used custody primitive in the ecosystem. On
Ethereum, Safe (formerly Gnosis Safe) is the dominant standard, deployed across
86 networks; on Solana, Squads is the analogue. Both are fully transparent by
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
shared control of protocol admin authorities. The launch-day Private Multisig
app (`A-multisig`) and the Private DAO (`LD-private-dao`) are blocked on it.

LEZ also makes it possible to close a gap no sovereign multisig has closed. As
the [appendix](../appendix/multisig-coordination-ecosystem.md) documents, every
existing implementation that keeps multisig structure private (FROST, MuSig2
n-of-n) does so by moving the quorum off-chain into a cryptographic session, at
the cost of hardware immaturity and, for MuSig2, an n-of-n-only limitation.
Every implementation that keeps the quorum on-chain is transparent. On LEZ, the
same multisig program can run over private accounts, so the chain records only a
commitment to the post-state and a validity proof, without a trusted coordinator
and without giving up k-of-n. This RFP is the vehicle for delivering that.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement an M-of-N multisig program on LEZ. A multisig is created with N
   members and a threshold M; an action requires at least M approvals before it
   can execute. Members, threshold, and the action set are configurable at
   creation.
2. Support the full proposal lifecycle: a member proposes an action; members
   approve or reject asynchronously; once M approvals are collected the action
   becomes executable; if rejections make M approvals impossible the proposal is
   dead. Support proposal cancellation before execution and proposal expiry.
3. Execute approved actions by delegating to any LEZ program via `ChainedCall`,
   following the existing `logos-co/lez-multisig` design: the proposal stores a
   serialised instruction and target program ID, and execution delivers it to
   the target. The multisig never modifies external state directly. See
   [Appendix section 6](../appendix/multisig-coordination-ecosystem.md#6-composition-how-a-multisig-triggers-actions).
4. Support configuration changes (add or remove a member, change the threshold)
   through the same M-of-N approval flow, so structural changes cannot bypass
   the quorum.
5. Support role separation among members so that the ability to propose, to
   approve, and to execute can be assigned independently, following the Squads
   permission model documented in
   [Appendix section 5](../appendix/multisig-coordination-ecosystem.md#5-roles-policies-and-extensibility).
   At minimum, a proposing key need not be an approving key.
6. Support an optional per-multisig time lock: a configurable delay between an
   action reaching quorum and becoming executable, enforced by the program. A
   time lock of zero means immediate execution.
7. Support an optional spending-limit policy: a member (or sub-quorum) may
   execute transfers up to a configured limit without the full M-of-N approval,
   per the policy model in
   [Appendix section 5](../appendix/multisig-coordination-ecosystem.md#5-roles-policies-and-extensibility).
8. Provision an end-to-end-encrypted coordination room per multisig using the
   Logos chat module (RD-chat), scoped to the multisig's members, for signer
   deliberation. See the Coordination Architecture section for what this channel
   carries and its privacy properties.
9. Support the privacy posture selected under Decision 1 (see Decisions for
   Review). The program must run the multisig over the account kind(s) required
   by that posture (public accounts, private accounts, or both) so that the data
   items designated private in Decision 1 are not published on-chain. See
   [Appendix section 1](../appendix/multisig-coordination-ecosystem.md#1-what-data-a-multisig-involves-and-whether-it-can-be-private)
   and
   [section 2](../appendix/multisig-coordination-ecosystem.md#2-the-lee-execution-model-public-vs-private-accounts).

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
   public and what is private under the selected privacy posture (which data
   items from Appendix section 1 are published on-chain and which are not), so a
   multisig operator understands exactly what an observer can see.
6. Before a member signs an approval, the mini-app must display the exact action
   that approval authorises (target program, decoded instruction, amounts), so
   the member verifies what they are signing rather than a UI-rendered summary.
   This directly addresses the signing-layer attack surface described in
   [Appendix section 1.2](../appendix/multisig-coordination-ecosystem.md#12-approval-attribution-and-pending-proposals-items-4-5).
7. Failed or rejected proposals and executions must return clear, actionable
   error messages.

#### Reliability

1. Multisig state must remain consistent under concurrent approvals from
   different members; no approval is lost or double-counted, and an action
   cannot execute with fewer than M valid approvals.
2. A configuration change (member set or threshold) must invalidate in-flight
   proposals whose approval set was gathered under the old configuration, unless
   they had already reached quorum, following the stale-proposal handling in
   [Appendix section 5](../appendix/multisig-coordination-ecosystem.md#5-roles-policies-and-extensibility).

#### Performance

1. Each operation (create, propose, approve, reject, execute) completes within a
   single LEZ transaction.
2. Compute unit usage and transaction size of each operation must be documented
   and benchmarked against LEZ devnet limits. If the selected privacy posture
   uses private-account execution, the per-proof cost and the per-block private
   transaction throughput must be measured and reported, since they bound how
   many approvals can be processed per block.

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
   approved once and executed atomically or in sequence, per the Squads batch
   model in
   [Appendix section 5](../appendix/multisig-coordination-ecosystem.md#5-roles-policies-and-extensibility).
2. Support weighted approvals, where members carry numeric weights and the
   threshold is a minimum cumulative weight, per
   [Appendix section 5](../appendix/multisig-coordination-ecosystem.md#5-roles-policies-and-extensibility).

### Out of Scope

The following are explicitly excluded from this RFP:

- A FROST or MuSig2 threshold-signature implementation. Structural privacy on
  LEZ is achieved through private-account execution (Appendix section 2), not
  through an off-chain threshold-signature session. Threshold-signature schemes
  are documented in the appendix as ecosystem context only.
- Recovery of a multisig whose members have lost quorum-many keys. Social or
  time-locked key recovery is a separate concern and is not required here.

### Coordination Architecture

Every multisig provisions one end-to-end-encrypted room using the Logos chat
module (RD-chat), scoped to its members. As the
[appendix](../appendix/multisig-coordination-ecosystem.md#4-coordination-how-signers-reach-quorum)
documents, no surveyed sovereign multisig offers an encrypted,
metadata-resistant coordination channel: coordination today is either public
on-chain state (Squads), a relay that sees the metadata (Safe), or a
user-supplied external channel (Bitcoin). The Logos chat module closes that gap.

What the room carries, and whether machine coordination (proposals and
approvals) also flows through it or stays as on-chain state, is the subject of
Decision 2 (see Decisions for Review). In all variants the room carries human
deliberation: the discussion among signers about whether to approve. The room is
coordination, not enforcement: quorum is always enforced by the program
on-chain, never by the chat channel.

### Privacy Architecture

A multisig involves at least ten distinct data and metadata items, enumerated in
[Appendix section 1](../appendix/multisig-coordination-ecosystem.md#1-what-data-a-multisig-involves-and-whether-it-can-be-private):
the existence of the multisig, the member set, the threshold, per-signer
approval attribution, pending-proposal metadata, the action payload, the vault
balance, execution linkage, coordination content, and the co-signing social
graph. On LEZ, each of these can independently be public or private, because the
Logos Execution Environment runs the same program over public accounts (visible
on-chain) or private accounts (only a post-state commitment and validity proof
on-chain), per
[Appendix section 2](../appendix/multisig-coordination-ecosystem.md#2-the-lee-execution-model-public-vs-private-accounts).

Which of these items must be private, which are left to the operator's choice,
and which are private by default with a public opt-out, is the subject of
Decision 1 (see Decisions for Review). Where identity is concerned, the property
in question is unlinkability between a user's account and its role in a
multisig, not concealment of the user. Documentation (Usability requirement U.5)
must state the resulting public/private split explicitly for the chosen posture.

## ⚠ Platform Dependencies

This RFP is open for proposals. Proposers may begin design and development work,
but a working on-chain deployment depends on the primitives below.

### Hard blockers

#### Private-account execution (core LEE feature)

If the selected privacy posture (Decision 1) requires any multisig data to be
private, the program relies on LEE private accounts: post-state commitments,
nullifiers, and Risc0 validity proofs, as described in
[Appendix section 2](../appendix/multisig-coordination-ecosystem.md#2-the-lee-execution-model-public-vs-private-accounts).
These are core LEE features rather than lambda prizes; proposers should confirm
the current state of private-account support on LEZ devnet against the Resources
below before relying on it, including whether program-derived private accounts
are supported for the multisig's vault.

#### Logos chat module (RD-chat)

The per-multisig coordination room (Functionality requirement F.8) depends on
the Logos chat module, tracked as `RD-chat`. Proposers should confirm the
module's availability and its SDK surface for creating a member-scoped,
end-to-end-encrypted room before relying on it.

### Resolved dependencies

#### Cross-program calls via ChainedCall

Execution delegates to a target program via `ChainedCall` (Functionality
requirement F.3). This is the mechanism the existing `logos-co/lez-multisig`
sample app already uses, delivered by the LEZ team as part of the core runtime.

### Risks

#### Private-transaction throughput

If the privacy posture uses private-account execution, each approval that
mutates private state produces a proof and consumes a private-transaction slot.
LEZ private-transaction throughput per block is limited (as of 2026-04, one
private transaction per block on the reference deployment); a multisig with many
members approving in quick succession may be bounded by this. Performance
requirement P.2 requires this to be measured and reported. If throughput is a
constraint, the proposal should describe how approvals are batched or
aggregated.

#### Signing-layer trust

As the appendix records, the Bybit and WazirX losses were not smart-contract
failures; they exploited the gap between what a signer saw in a UI and what they
actually authorised. Usability requirement U.6 (show the exact decoded action
before signing) is the primary mitigation and must not be treated as optional
polish.

## 👤 Recommended Team Profile

Team experienced with:

- Multisig, threshold custody, or account-abstraction design
- Rust program development for a RISC-V or zkVM target (Risc0 experience a plus)
- Applied cryptography and zero-knowledge proof systems
- Secure signing UX and hardware-wallet integration
- Front-end development for custody or wallet applications

## ⏱ Timeline Expectations

Estimated duration: **12 weeks** (production-hardening of the proven
`lez-multisig` design, plus the coordination room and the selected privacy
posture). The estimate assumes the privacy posture is settled at contracting
time via Decision 1; a posture requiring private-account execution of the full
proposal lifecycle sits at the upper end of the range.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [logos-co/lez-multisig](https://github.com/logos-co/lez-multisig): the
  existing public multisig sample app this RFP hardens
- [Introduction to the Logos Execution Zone](https://docs.logos.co/lez): the
  public/private account model this RFP relies on

## 🧩 Decisions for Review

The following three decisions shape the deliverable and should be settled before
contracting. Each is presented as options A / B / C with the trade-offs drawn
from the [appendix](../appendix/multisig-coordination-ecosystem.md). A
recommended default is marked, but the choice is open.

### Decision 1 — Privacy posture: which multisig data is private?

Per Appendix section 1, a multisig involves ten distinct data items, each of
which can independently be public or private on LEZ. This decision sets, per
item, whether privacy is mandatory, operator-selectable, or default-on.

- **Option A — Public multisig (transparent, like Safe/Squads).** All ten data
  items are public. Simplest to build and audit; matches the existing
  `lez-multisig` sample app; gives the transparent, publicly-inspectable
  behaviour some treasuries and DAOs want. But it carries the same
  transparency-as-attack- surface profile as Safe and Squads, and delivers none
  of the LEZ privacy advantage.
- **Option B — Operator choice per multisig (recommended default).** The program
  supports both public and private execution; at creation the operator chooses
  the posture. A recommended default configuration is shipped (see below).
  Maximises fitness across use cases (a transparent DAO treasury and a private
  company multisig from the same program) at the cost of building and testing
  both execution paths. Recommended default within this option: existence,
  member set, approval attribution, action, execution linkage, and social graph
  **private**; coordination content **always private** (E2EE room); threshold,
  pending-proposal visibility, and holdings **operator-selectable**.
- **Option C — Private-by-default multisig.** All items that can be private are
  private by default, with a per-item public opt-out. Leans hardest into the LEZ
  differentiator and gives the strongest default privacy, but every deployment
  pays the private-execution cost (proof generation, throughput limits per
  Appendix section 2 and the Risk above) even where an operator did not need it.

### Decision 2 — Coordination channel: where do approvals flow?

Per Appendix sections 1.2 and 4, machine coordination (proposals and approvals)
can live as on-chain state or off-chain, while human deliberation is always
off-chain. This decision sets where each flows. In all options, quorum is
enforced on-chain by the program.

- **Option A — On-chain approvals + E2EE deliberation room (recommended
  default).** Proposals and approvals are program state on-chain (public, or
  private per Decision 1); the E2EE room carries human deliberation only. Keeps
  a clean, program-enforced approval record and a live pending view, exactly as
  `lez-multisig` does today, while adding the encrypted discussion channel no
  competitor offers. Note (per Appendix section 1.2) this on-chain record is not
  an audit-trail advantage over off-chain collection; both yield a record at
  execution. Its value here is a simple, verifiable machine channel.
- **Option B — Approvals carried through the E2EE room.** The room is both the
  deliberation channel and the transport for signing material; the program
  verifies the collected approvals at execution (the Safe off-chain-collection
  shape, but over an encrypted, metadata-resistant channel). Removes
  per-approval on-chain writes and hides pending-proposal metadata even in a
  public posture, at the cost of a more complex client that must reliably gather
  and present approvals, and no live on-chain pending view.
- **Option C — Both, operator-selectable.** Ship both channels and let the
  operator choose per multisig. Most flexible; largest client surface to build
  and test.

### Decision 3 — Quorum model baseline

Per Appendix sections 2 and 3, structural privacy on LEZ comes from
private-account execution of an account-model multisig, not from a
threshold-signature scheme. This decision confirms the baseline the program is
built on.

- **Option A — Account-model quorum on LEZ, hardening `lez-multisig`
  (recommended default).** Members and threshold are program state; approvals
  are program-verified; privacy (if selected) comes from running over private
  accounts. Direct continuation of the proven sample app; the appendix shows
  this is the only surveyed path to private, coordinator-free, k-of-n multisig.
- **Option B — Threshold signatures (FROST-style), single aggregate signature
  on-chain.** Maximal structural privacy at the signature layer, but the
  appendix documents FROST as BIP-draft, hardware-immature, and (for MuSig2)
  n-of-n only; this is why it is Out of Scope above. Listed for completeness;
  not recommended.
- **Option C — Deliver the account model now, keep a threshold-signature variant
  as a documented future extension.** Build Option A, and specify the interface
  seam where a threshold-signature signing path could be added later without
  redesign.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
