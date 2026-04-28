---
id: RFP-000
title: Project Title
tier: XS/S/M/L/XL
funding: $XXXXX
status: open/closed
category: Developer Tooling & Infrastructure / Applications & Integrations / Ecosystem & Community Enablement
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-000 — Project Title

## 🧭 Overview

Briefly describe the opportunity in 4–6 sentences.

Explain things like:

- What needs to be built
- Why it matters
- How it strengthens the Logos ecosystem
- The type of team likely to succeed


## 🔥 Why This Matters

Provide ecosystem context in 4-6 sentences.

Explain things like:

- Why this RFP is critical in the success of the Logos ecosystem
- How can this RFP unlock user or developer adoption  
- How can this RFP enable liquidity  
- How can this RFP improve privacy guarantees  
- How can this RFP reduce technical barriers

Builders want to understand the impact of th RFP towards the success of Logos ecosystem.


## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

List what the program/application must do.

#### Usability

Standard requirements for Logos apps (adapt as needed):

1. Provide an SDK that can be used to build Logos modules for
   interacting with the program.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations.
4. Provide an IDL for the LEZ program, preferably using the
   [SPEL framework](https://github.com/logos-co/spel).

Add RFP-specific usability requirements here.

#### Reliability

List reliability guarantees (consistency, fault tolerance, graceful
degradation, etc.).

#### Performance

List performance requirements. Document compute unit usage of each
operation (LEZ's per-transaction compute budget may change during
testnet).

#### Supportability

Standard requirements (adapt as needed):

1. The program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone
   mode) and are included in CI
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
5. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for interacting with the
   program via CLI and mini-app.
6. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the Logos module/library, covering the developer integration journey.
7. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
8. Provide Figma designs or equivalent for all GUI artifacts.
9. Provide a privacy and anonymisation properties document that
   addresses every scenario listed in the Threat Model & Security
   Requirements section. For each scenario, the document must state
   how it is defended (test reference, formal argument, or program
   invariant) or explicitly mark it out of scope with rationale.

### Soft Requirements

Explain all the optional soft requirements of the RFP in points.


## 🛡️ Threat Model & Security Requirements

This section enumerates the adversaries and scenarios the implementation
must protect against. Each numbered scenario is a testable hard
requirement. The proposal must respond to each scenario with one of:

- a corresponding test or formal argument that the scenario is
  defended,
- an explicit out-of-scope acknowledgement with rationale, or
- a documented trust assumption inherited from the Logos stack (see
  below).

Supportability item 9 (privacy and anonymisation properties document)
must address every scenario listed here in the same form.

The threat model is scoped to what the delivered application stack
(on-chain program, SDK, mini-app) controls. Properties of the
underlying Logos platform are inherited as trust assumptions and are
not the responsibility of this RFP.

### Trust assumptions inherited from the Logos stack

The proposal inherits the trust assumptions documented in
[`appendix/logos-stack-trust-assumptions.md`](../appendix/logos-stack-trust-assumptions.md).
That appendix is a dated snapshot of the
[stack FURPS](https://roadmap.logos.co) and lists both what an RFP can
rely on (programmable privacy, sequencer behaviour, indexer
correctness, blockchain finality, block proposer privacy) and what an
RFP must not assume (anonymous transaction submission at the network
layer, mempool privacy, anonymous RPC queries, off-chain storage
privacy for LEZ state, sequencer-level censorship resistance, side
channels).

The proposal must not claim to defend against weaknesses in the
inherited assumptions. The proposal must not claim to inherit a
guarantee that the snapshot does not commit to.

If the proposal depends on a specific FURPS item, it must cite the
item by component and number (e.g., "relies on
`lez:programmable-privacy.22`") so that reviewers can verify the
dependency against the snapshot.

### Adversary classes

Scenarios are grouped by adversary class and identified by a stable
code (e.g., `O-1`) so other documents and reviews can reference them.
RFP authors must keep the common scenarios verbatim and add
protocol-specific scenarios within the relevant adversary class,
continuing the numbering.

- **O. On-chain observer.** Reads all public on-chain state and the
  full transaction history touching the program. Also sees any
  user-correlatable patterns the SDK or mini-app produce through RPC
  and indexer queries, since the application chooses those query
  patterns. Does not submit transactions.
- **A. Active on-chain attacker.** Submits arbitrary transactions,
  deploys other programs, pays priority fees, and controls multiple
  addresses. Tries to break the program's invariants.
- **C. Malicious counterparty.** A user the victim transacts with
  directly through the program: trade counterparty, LP, borrower,
  lender, launchpad participant, swap maker.
- **S. Sequencer with elevated visibility.** The LEZ sequencer
  receives, orders, and posts user transactions. The stack does not
  commit to non-censorship or to non-correlation against the
  sequencer. The application controls what it reveals to the
  sequencer beyond what is unavoidable, and what its safety depends
  on.
- **F. Malicious or buggy client.** A user runs a hostile mini-app, a
  third-party SDK, or hand-crafted transactions that bypass the
  official SDK.
- **X. External identity correlator.** Holds off-chain data (CEX KYC,
  IP logs, social graph, public wallet labels) and tries to link it
  to on-chain activity.

### Common scenarios

The following scenarios apply to every Logos RFP that supports private
account interaction. Each implementation must protect against them or
explicitly document them as out of scope or inherited.

#### O. On-chain observer

1. **O-1.** Given a series of operations executed via the
   deshield→interact→re-shield pattern, the observer cannot determine
   which private account originated the funds for any operation from
   on-chain state alone.
2. **O-2.** Given two operations from the same private-account user,
   the observer cannot link them on-chain. The program does not store
   any field that joins two ephemeral accounts to a common owner.
3. **O-3.** Given an ephemeral public account used for one operation,
   the observer cannot link it to the destination private account
   that receives the re-shielded output.
4. **O-4.** The program's account layout, transaction structure, and
   event emissions do not include identifiers, salts, or correlatable
   metadata that would let an observer cluster operations by
   originating private account.
5. **O-5.** The SDK and mini-app do not issue RPC or indexer queries
   that re-link ephemeral accounts to a common owner (e.g., a
   batched lookup that names multiple ephemeral accounts in a
   single request, or a query that pairs an ephemeral account with
   the user's private-account identifier). RPC-level anonymity is
   not inherited from the stack; the application minimises leakage
   through call patterns.

#### A. Active on-chain attacker

1. **A-1.** The attacker cannot drain protocol funds, mint phantom
   positions, or corrupt protocol state via reentrancy, race
   conditions, compute-budget exhaustion, or arithmetic overflow.
2. **A-2.** The attacker cannot bypass authority checks (admin,
   freeze, role-gated operations) by forging signatures, replaying
   transactions, or substituting accounts.
3. **A-3.** The attacker cannot front-run, sandwich, or otherwise
   target a victim by linking the victim's on-chain identity to a
   transaction. Identity-linked attacks (wallet profiling, repeated
   targeting of the same victim) must be impossible when the victim
   uses the deshield→interact→re-shield pattern. Size-based or
   pool-state-based attacks (sandwiching, back-running) that do not
   rely on identity are out of scope unless the RFP explicitly states
   otherwise.

#### C. Malicious counterparty

1. **C-1.** A counterparty cannot steal funds mid-settlement. Every
   protocol-mediated exchange must complete atomically or fully
   revert.
2. **C-2.** A counterparty cannot grief the user by leaving funds
   locked, stranded in an ephemeral account, or recoverable only via
   privileged intervention.

#### S. Sequencer with elevated visibility

1. **S-1.** Program safety must not depend on the sequencer behaving
   honestly. A malicious or compromised sequencer that censors,
   reorders, or delays transactions can cause liveness loss but
   cannot mint, steal, or corrupt protocol state.
2. **S-2.** Information the sequencer learns from a user's
   transaction submission (transaction contents, the user's network
   identity at submission time, pre-confirmation ordering) does not
   exceed what is already visible on-chain. The program does not
   place additional user-correlatable metadata in fields the
   sequencer reads but the chain does not commit.

#### F. Malicious or buggy client

1. **F-1.** The on-chain program rejects any transaction that
   violates protocol invariants regardless of how the transaction is
   constructed. Integrity guarantees do not depend on SDK or
   mini-app correctness. Where privacy depends on client behaviour
   (e.g., the deshield→interact→re-shield pattern), the proposal
   states this explicitly and the SDK enforces the pattern as a
   single indivisible user action.
2. **F-2.** The SDK rejects privacy-breaking inputs by construction:
   re-shielding to a public account, reusing an ephemeral account,
   funding an ephemeral account from an external source, or
   splitting a single conceptual operation across multiple
   user-visible signing steps.

#### X. External identity correlator

1. **X-1.** Funding the ephemeral public account from any source
   other than the atomic deshield (e.g., a CEX withdrawal, a
   transfer from an existing wallet) creates an on-chain link to a
   known identity and breaks the privacy guarantee. The SDK makes
   external funding impossible by construction. The deshield
   (operation token plus gas) is a single indivisible user action.
2. **X-2.** The mini-app and documentation make the privacy
   boundaries explicit, identifying which user actions preserve
   unlinkability and which actions break it (re-shielding to a
   known wallet, sending re-shielded funds to a CEX, signing with a
   publicly attested key).

### Protocol-specific scenarios

RFP authors must append protocol-specific scenarios under the
relevant adversary class, continuing the numbering. Use the same
format: a concrete adversary capability, a clear scenario, and the
testable defence.

Examples of where protocol-specific scenarios commonly apply:

- Oracle-dependent protocols (lending, perpetuals): A-class
  scenarios for oracle manipulation and liquidation-driven MEV.
- Launchpad and auction protocols: C-class scenarios for griefing
  the price discovery process; F-class scenarios for sniping
  protections.
- DEXes: A-class scenarios that document the boundary between
  identity-linked attacks (in scope) and pool-state attacks (out of
  scope by default).
- Protocols sensitive to ordering: S-class scenarios documenting
  whether the program's correctness depends on sequencer ordering
  or only on chain-finalised order.

Scenarios that the proposal treats as out of scope (e.g.,
side-channel correlation via transaction timing, sandwich attacks
based purely on pool state, network-layer observability of
transaction submission, key-loss recovery) must still be listed with
explicit rationale rather than omitted.


## 👤 Recommended Team Profile

Signal the bar without gatekeeping. Mention all the areas where the applying should ideally have experience with. Examples:

- Distributed systems  
- Cryptography  
- Wallet infrastructure  
- Production deployments


## ⏱ Timeline Expectations

Provide a realistic range aligned with tier. Example: Estimated duration: **3–7 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 License**.


## Resources

List all the relevant resources that can be useful to understand more about the RFP/related work.


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use **Discussions**.
