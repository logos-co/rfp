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

This section enumerates the adversaries and scenarios the
implementation must protect against. Each numbered scenario is a
testable hard requirement. The proposal must respond to each scenario
with one of:

- a corresponding test or formal argument that the scenario is
  defended,
- an explicit out-of-scope acknowledgement with rationale, or
- a documented trust assumption inherited from the Logos stack (see
  below).

Supportability item 9 (privacy and anonymisation properties document)
must address every scenario listed here in the same form.

The threat model is scoped to what the delivered application stack
(programs, modules, SDKs, mini-app) controls. Properties of the
underlying Logos platform are inherited as trust assumptions.

### Stack components consumed

The proposal must declare which Logos stack components the application
consumes. The relevant component-specific scenarios then apply in
addition to the common scenarios. Tick all that apply:

- [ ] **Blockchain / LEZ**: programs, sequencer, indexer, wallet,
  bridging.
- [ ] **AnonComms**: Mix transport, RLN identity service.
- [ ] **Logos Core**: module loader, QObject Registry, Process
  Manager.
- [ ] **Messaging (Waku)**: Relay, Light Push, Store, RLN Relay,
  messaging SDK.
- [ ] **Storage (Codex)**: privacy-preserving filesharing,
  anonymous downloads.

### Trust assumptions inherited from the Logos stack

The proposal inherits the trust assumptions documented in
[`appendix/logos-stack-trust-assumptions.md`](../appendix/logos-stack-trust-assumptions.md).
That appendix is a dated snapshot of the
[stack FURPS](https://roadmap.logos.co) and lists, per component, both
what an RFP can rely on and what an RFP must not assume.

The proposal must not claim to defend against weaknesses in the
inherited assumptions. The proposal must not claim to inherit a
guarantee that the snapshot does not commit to.

If the proposal depends on a specific FURPS item, it must cite the
item by component and number (e.g., "relies on
`lez:programmable-privacy.22`" or "relies on `messaging:rln_relay:F.3`")
so that reviewers can verify the dependency against the snapshot.

### Adversary classes

Scenarios are grouped by adversary class and identified by a stable
code (e.g., `N-1`) so other documents and reviews can reference them.
RFP authors must keep the common scenarios verbatim and add
component-specific and protocol-specific scenarios within the
relevant adversary class, continuing the numbering.

- **N. Network observer.** Reads everything visible on the public
  surfaces the application touches: on-chain state and transactions;
  Waku messages, topics, and metadata; Codex content delivery and DHT
  queries; indexer and RPC traffic; client-issued query patterns. Does
  not actively interfere.
- **A. Active attacker.** Submits transactions, messages, queries,
  uploads. Deploys programs, runs malicious peer nodes (Waku store or
  relay, Codex cache or provider, RPC endpoints, sequencer-attached
  infra). Pays priority fees. Controls multiple identities. Tries to
  break protocol invariants or coerce malicious responses.
- **C. Malicious counterparty.** A user or peer the victim transacts,
  communicates, or exchanges with directly through the application:
  trade counterparty, LP, lender, launchpad participant, chat peer,
  swap maker, content publisher.
- **P. Platform operator with elevated visibility.** A node or
  operator the application routes through and trusts for liveness
  only: LEZ sequencer; Waku store, relay, or RLNaaS provider; Codex
  cache or provider; indexer or RPC operator. The stack does not
  commit to non-correlation against these operators; the application
  chooses what it reveals to them and what it depends on them for.
- **F. Malicious or buggy client.** A user runs a hostile mini-app, a
  third-party SDK, or hand-crafted requests that bypass the official
  client.
- **X. External identity correlator.** Holds off-chain data (CEX KYC,
  IP logs, social graph, public address labels, leaked address books)
  and tries to link it to application activity.

### Common scenarios

The following scenarios apply to every Logos RFP regardless of which
components it uses. Each implementation must protect against them or
explicitly document them as out of scope or inherited.

#### N. Network observer

1. **N-1.** Public surfaces produced by the application (on-chain
   transactions, Waku messages, Codex content and queries, indexer
   and RPC traffic) do not contain user-correlatable identifiers,
   salts, or metadata that allow an observer to cluster a user's
   operations beyond what the underlying primitive intentionally
   exposes.
2. **N-2.** The client and SDK do not issue queries that re-link
   user activity across operations: no batched RPC over multiple
   ephemeral identities in one request, no Waku store query that
   pairs ephemeral and persistent topics, no Codex DHT query that
   names a sequence of CIDs from one user.
3. **N-3.** Where the application composes a private-state container
   (LEZ private account, encrypted Waku topic, end-to-end encrypted
   Codex blob) with a public-action surface, the transition between
   them is enforced as an indivisible user action. Seeding the
   public surface from external sources (a known wallet, a public
   peer identity, a long-lived RLN membership) is impossible via
   the official SDK.
4. **N-4.** Persistent identifiers the application issues (LEZ
   public account addresses, Waku content topics, Codex CIDs, RLN
   memberships) do not encode information that links them to a
   user's other identifiers across the stack.

#### A. Active attacker

1. **A-1.** The attacker cannot violate protocol invariants
   regardless of how a transaction, message, query, or upload is
   constructed. Authoritative validation (on-chain program,
   server-side validator, peer-side verifier) rejects all invalid
   state transitions.
2. **A-2.** The attacker cannot bypass authority or access checks
   (admin roles, RLN membership, encryption-key gates,
   capability-based permissions) by forging, replaying, or
   substituting credentials.
3. **A-3.** The attacker cannot identity-link a target through
   patterns the application could prevent: user fingerprints in
   transaction structure, peer identifiers in Waku metadata,
   address fingerprints in Codex DHT lookups, repeated client
   fingerprints in RPC traffic. Attacks that depend on properties
   the application does not control (raw network position, timing
   visible at the transport layer) must be marked inherited or out
   of scope.

#### C. Malicious counterparty

1. **C-1.** Where the application mediates an exchange (trade,
   swap, settled message, asset transfer, file delivery), a
   counterparty cannot steal mid-settlement. The exchange completes
   atomically or fully reverts.
2. **C-2.** A counterparty cannot grief the user by stranding their
   state: funds locked in an ephemeral account, content stuck
   mid-upload, messages in a deadlocked group, cryptographic
   capability lost, recoverable only via privileged intervention.

#### P. Platform operator with elevated visibility

1. **P-1.** Application safety does not depend on the honesty of
   any platform operator. A malicious or compromised operator (LEZ
   sequencer, Waku store or relay, Codex cache or provider,
   RPC endpoint, RLNaaS provider) can cause liveness loss but
   cannot mint, steal, corrupt protocol state, or break the
   application's documented privacy guarantees.
2. **P-2.** The application reveals to each platform operator only
   what is unavoidable for that operator's role. Metadata the
   application could withhold (correlatable salts, batched
   queries, payload framing, identity hints) is withheld.

#### F. Malicious or buggy client

1. **F-1.** Authoritative components (on-chain programs, server-side
   validators, peer-side verifiers) reject invariant-violating
   inputs regardless of how they are constructed. Integrity
   guarantees do not depend on client correctness.
2. **F-2.** Where privacy depends on client behaviour, the SDK
   enforces the privacy-preserving pattern as a single indivisible
   user action and rejects privacy-breaking inputs by construction
   (e.g., funding an ephemeral account from a known wallet, reusing
   a single-use Waku topic, decrypting and republishing a Codex
   blob, splitting a conceptual operation across multiple
   user-visible signing steps).

#### X. External identity correlator

1. **X-1.** The application makes external linkage difficult by
   construction. Where the user crosses a privacy boundary
   (re-shielding to a known wallet, sending to a CEX, posting under
   a publicly-attested key, uploading content from a known
   publisher identity), the SDK and mini-app warn explicitly before
   the action.
2. **X-2.** Documentation enumerates which user actions preserve
   unlinkability and which actions break it, in user-visible
   language.

### Component-specific scenarios

The following scenarios apply only when the corresponding component
is declared in the "Stack components consumed" section above. RFP
authors must include the scenarios for each declared component
verbatim and may extend them with protocol-specific items
(continuing the numbering).

#### Blockchain / LEZ

Apply when the application runs an LEZ program or interacts with
LEZ accounts.

1. **L-1.** Operations executed via the deshield→interact→re-shield
   pattern do not link the originating private account to any
   on-chain artefact. The program stores no field that joins two
   ephemeral accounts to a common owner.
2. **L-2.** Ephemeral public accounts created during a
   deshield→interact→re-shield flow are single-use. The SDK does
   not reuse them across operations.
3. **L-3.** Where a deshield→interact→re-shield flow is required for
   privacy, the deshield is a single indivisible user action that
   transfers both the operation token and the gas required for the
   downstream interaction in one step.
4. **L-4.** The SDK validates that the re-shield destination is a
   private (shielded) account before submitting the transaction and
   rejects with an explicit error if it is not.
5. **L-5.** Program correctness does not rely on the LEZ sequencer's
   honesty. A malicious or compromised sequencer that censors,
   reorders, or delays transactions can cause liveness loss but
   cannot mint, steal, or corrupt protocol state. Cite
   `lez:sequencer.23-25` for the inherited liveness role.
6. **L-6.** SDK and mini-app queries to the LEZ indexer RPC do not
   pair multiple ephemeral accounts in a single request and do not
   batch ephemeral and persistent identifiers in a way that re-links
   them.

#### AnonComms

Apply when the application explicitly composes Mix or RLN.

1. **AC-1.** Where the application routes traffic through Mix, it
   does so via the documented integration points (the libp2p mix
   protocol, Logos Core mix module, or Waku Lightpush mix path) and
   does not bypass them on a fallback path that lacks the same
   protections.
2. **AC-2.** RLN memberships obtained through the AnonComms RLN
   service are not reused across application sessions where
   unlinkability is required between sessions.

#### Logos Core

Apply when the application ships as a Logos Core module.

1. **LC-1.** The module fails closed on platform errors. A failing
   QObject Registry lookup, a crashed sibling module, or a stalled
   Process Manager does not cause the module to bypass
   authority checks or expose user state.
2. **LC-2.** The module declares its inter-module dependencies and
   does not assume the presence of capabilities (transport, UI,
   networking) that Logos Core does not itself provide.

#### Messaging (Waku)

Apply when the application uses Waku for any user-facing
communication.

1. **W-1.** Content topics chosen by the SDK do not encode
   user-identifying information. Ephemeral topics used for
   unlinkable interactions are not reused across user sessions.
2. **W-2.** RLN proofs attached to outbound messages do not leak
   identity beyond the rate-limit guarantee. The application does
   not include user-correlatable metadata in message envelopes that
   the RLN proof would otherwise protect.
3. **W-3.** Waku store queries do not pair ephemeral and persistent
   topics in a single request. The SDK pages and time-windows store
   queries to limit what a store node operator can correlate.
4. **W-4.** Application safety does not depend on the chosen Light
   Push or RLNaaS provider's honesty. Provider failure causes
   liveness loss only.

#### Storage (Codex)

Apply when the application uses Codex for content distribution or
archival.

1. **CX-1.** Codex uploads from the application do not include
   metadata that links them to a known publisher identity. Where
   the application's privacy model requires publisher
   unlinkability, the upload path uses the privacy-preserving
   variant rather than a direct provider upload.
2. **CX-2.** Downloads use the anonymous-downloads-over-mix path
   when the application's privacy model requires downloader
   unlinkability. The SDK does not query for the same content via
   two distinguishable downloader identities in a way that
   re-links a single user.
3. **CX-3.** Where the application caches Codex content, it does so
   in a form that supports the cache plausible-deniability property
   declared in the Storage FURPS (the cache does not store
   plaintext or otherwise gain knowledge of the contents it caches).

### Protocol-specific scenarios

RFP authors must append protocol-specific scenarios under the
relevant adversary class or component-specific section, continuing
the numbering. Use the same format: a concrete adversary capability,
a clear scenario, and the testable defence.

Examples of where protocol-specific scenarios commonly apply:

- Oracle-dependent protocols (lending, perpetuals): A-class
  scenarios for oracle manipulation and liquidation-driven MEV.
- Launchpad and auction protocols: C-class scenarios for griefing
  the price discovery process; F-class scenarios for sniping
  protections.
- DEXes: A-class scenarios that document the boundary between
  identity-linked attacks (in scope) and pool-state attacks (out of
  scope by default).
- Group messaging or forum protocols: C-class scenarios for
  member-join griefing; W-class scenarios for membership-change
  unlinkability.
- Filesharing and archival protocols: CX-class scenarios for
  publisher anonymity sets and download cover-traffic.
- Protocols sensitive to ordering: P-class scenarios documenting
  whether correctness depends on a specific operator's ordering.

Scenarios the proposal treats as out of scope (e.g., side-channel
correlation via transaction timing, sandwich attacks based purely
on pool state, network-layer observability of transaction
submission, key-loss recovery) must still be listed with explicit
rationale rather than omitted.


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
