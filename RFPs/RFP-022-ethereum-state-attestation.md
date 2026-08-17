---
id: RFP-022
title: Trustless Ethereum State Attestation for LEZ
tier: M
status: open
category: Developer Tooling & Infrastructure
dependencies:
  - id: RFP-001
    reason: Admin authority governs the weak-subjectivity checkpoint, the chain registry, and the finality parameters, as specified in Functionality.
  - id: LP-0012
    reason: Structured on-chain events let clients and off-chain components follow committee handoffs and attestation submissions without polling every account.
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-022 — Trustless Ethereum State Attestation for LEZ

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) for complying with any legal and regulatory requirements that apply
> to them. Software developed under the Program is published and maintained by
> its developers, not by Logos.
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
> does not make any representation, provides any advice or assumes any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

Build a reusable LEZ primitive that lets any LEZ program act on a fact about
Ethereum state and know that fact is true, with no trusted intermediary.

A LEZ execution is self-contained: a program reads inputs from LEZ accounts and
instruction data, runs in the zkVM, and produces a proof that the execution was
correct **given those inputs**. Whether an input is a true statement about the
outside world is not something that proof covers. A program that mints against
"a deposit of amount X occurred on Ethereum" proves only that it minted given
the claim; the claim itself is uncertified, and whoever controls the input can
assert a deposit that never happened.

This RFP delivers the missing piece: a LEZ module that verifies Ethereum
consensus and finality, verifies inclusion of a referenced piece of state under
a finalised Ethereum header, evaluates a predicate over that state, and emits a
**verified statement** with no application action attached. It takes an Ethereum
header, the consensus material needed to prove that header is finalised, an
inclusion proof, and a predicate to check. It returns a proven assertion that
any LEZ program can consume.

The deliverable is deliberately application-free. Minting, escrow release,
settlement, and every other action belong to the consuming program. This RFP
delivers only the attestation, plus the tooling and documentation a consumer
needs to integrate it.

Teams will need experience with Ethereum consensus and the light-client
protocol, Merkle-Patricia proof verification, and LEZ program development inside
the RISC0 zkVM.

## 🔥 Why This Matters

Every LEZ program that wants to react to something that happened on Ethereum
faces the same problem, and today each would have to solve it independently.
Three workstreams already need it, and they need the same thing:

- **The wrapped-token bridge** ([RFP-021](./RFP-021-wrapped-erc20.md)). An
  Ethereum vault escrows an ERC-20 or native ETH and a LEZ program mints a
  canonical wrapped representation. Minting must happen only against a real,
  finalised deposit, with no signer or federation trusted to attest to it. That
  is a verified deposit statement feeding a mint. RFP-021 already requires
  trustless verification of Ethereum consensus and state rather than an
  inclusion check alone; this RFP is where that verification is specified and
  built.
- **Private cross-chain swaps.** A cross-chain swap that settles on the users'
  home chains, with no liquidity resting on LEZ, needs LEZ to confirm that both
  legs deposited before it authorises release. Confirming the Ethereum leg is a
  verified escrow statement, with no mint and no wrapped asset created. This is
  the read-only consumer, using the same attestation with no minting action
  attached.
- **The native gas token bridge** ([RFP-023](./RFP-023-gas-token-bridge.md)).
  The reverse-direction counterpart to RFP-021, in which LEZ is the vault and
  Ethereum the minter. Releasing native gas token from the LEZ vault requires
  proof that the corresponding ERC-20 was really burned on Ethereum, which is
  another verified statement over finalised Ethereum state, consumed by a
  program doing something different again with it.

These consumers do different things with the statement and carry different trust
and compliance surfaces, but they rely on the same attestation. That is the case
for building it once, as a shared primitive with a single audited verification
path, rather than separately inside each application. It also means the hardest
and most security-critical part of a cross-chain design (consensus verification)
gets one audit, one test suite, and one set of documented trust assumptions,
instead of one per consumer.

Building it once has a second consequence worth stating: it lowers the barrier
for any future LEZ program that wants to read Ethereum. Cross-chain lending
collateral checks, governance actions mirrored from an Ethereum DAO, proof of
holding an Ethereum-side asset, and airdrop eligibility derived from historic
Ethereum state all reduce to "evaluate a predicate over finalised Ethereum
state", which is exactly what this primitive returns.

## 🏗 Design Rationale

### Why an inclusion proof alone is not enough

The tempting shortcut is to verify an inclusion proof: supply a state root or
block header along with a Merkle path showing the state is included under that
root, and check the path inside the zkVM. That step is necessary and it is not
sufficient.

Given a chosen state, anyone can construct a header and a set of Merkle paths
that are internally consistent with it. An inclusion proof verifies that a leaf
is included under a **supplied** root. Whether that supplied root is the root of
the real Ethereum chain stays unchecked. A party free to choose the root can
fabricate a self-consistent state in which any deposit they like appears to
exist, generate a valid inclusion proof against it, and pass verification. The
zkVM proof will be sound. It will be proving a statement about a fabricated
Ethereum.

The missing ingredient is an anchor to canonical Ethereum. Verification has to
establish that the header it reasons about is a header the Ethereum network
agreed on, before any inclusion check against it carries meaning.

### The anchor: Ethereum finality via the sync committee

The anchor comes from Ethereum finality. Under proof of stake, Ethereum
finalises checkpoints: once a checkpoint is justified and then finalised,
roughly two epochs after a block, reverting it requires a consensus failure
severe enough to slash at least a third of all staked ETH. The light-client
protocol exposes this through finality updates, and a sync committee of 512
validators, rotating every sync-committee period, signs headers so a light
client can follow the finalised chain by checking committee signatures. The
beacon state holds `current_sync_committee` and `next_sync_committee`, so the
next committee is known a period ahead and handoffs can be followed forward. A
party tracking the sync committee can confirm that a given header is a finalised
header of the canonical chain without running a full node.

An attestation therefore establishes three things, in this order:

1. **Finality.** The supplied Ethereum header belongs to the finalised set,
   verified against sync-committee signatures over the light-client finality
   update. This is the basis the inclusion step depends on.
2. **Inclusion.** The referenced state or event (a deposit, a contract storage
   slot, a log) is included under that finalised header's state or receipts
   root, verified by Merkle proof.
3. **Predicate.** A statement over that verified state holds, for example
   "contract C holds an escrow of amount X in token T".

The output is a compact, proven assertion that the predicate holds of finalised
Ethereum state, carrying no trust in whoever generated the proof. A dishonest
prover cannot forge it, since forging it would require forging sync-committee
signatures over a finalised header. A dishonest prover can only decline to
produce a proof, which is why proof generation must be a liveness role rather
than a trust role: no single party may be able to turn a refusal into a block on
an honest user.

### Trust assumptions the design carries

The anchor is not free of assumptions, and both must be documented for consumers
rather than buried:

- **Honest sync-committee supermajority.** Following the chain through committee
  signatures trusts that at least two thirds of the current 512-validator sync
  committee are honest. This is weaker than Ethereum's full economic finality: a
  committee that colludes or is compromised could sign a header off the
  canonical chain, an expensive but real attack surface that verifying the full
  validator set would avoid. It is the standard light-client trade-off, and the
  proposal must state it plainly and quantify the cost of the attack it admits.
- **Weak-subjectivity checkpoint.** Sync-committee tracking only works starting
  from a trusted checkpoint, a known-good committee, following handoffs forward
  from there. That initial checkpoint is a trust anchor. The design is trustless
  after bootstrap, given a correct starting committee, so the specification must
  say who supplies the checkpoint, how a verifier confirms it independently, and
  how it is refreshed if a deployment falls outside the weak-subjectivity
  period.

Applicants may propose a stronger anchor than sync-committee tracking (full
validator-set verification, or a hybrid) and are encouraged to state the cost
trade-off explicitly if they do. The sync-committee path is the expected
baseline because it is what the light-client protocol supports today at
tractable in-zkVM cost.

### Chain binding

The attestation carries which chain and which chain ID it refers to, so a
statement valid for one network is never accepted as valid for another. This is
not incidental: the same contract address exists on mainnet and on every
testnet, and a design that omits the binding would let a testnet deposit satisfy
a mainnet predicate.

### Uniqueness is the consumer's job, not the primitive's

Two sub-problems sit inside "act on an Ethereum fact":

1. **Existence.** Did the referenced Ethereum state actually occur on the
   canonical Ethereum chain?
2. **Uniqueness.** Has this particular piece of Ethereum state already been
   consumed on LEZ, so that it cannot be acted on twice?

This RFP delivers existence. Uniqueness is solvable with mechanisms LEZ already
has: a consuming program records a nullifier or a program-owned account marking
a given Ethereum event as spent and rejects any second attempt. That is standard
double-spend protection, and it belongs to the consumer because only the
consumer knows what "the same action" means for its own application: a bridge
mint must be once-per-deposit, whereas a read-only escrow check may legitimately
be evaluated many times.

The primitive must nonetheless make uniqueness enforceable. Every verified
statement carries a deterministic, collision-resistant identifier derived from
the attested state itself, so a consumer can key a nullifier on it without
inventing its own derivation and without two consumers deriving incompatible
identifiers from the same event. Guidance on doing this correctly is a
documentation deliverable (Supportability #6).

### What the attestation does not do

**It does not hide the underlying Ethereum activity.** Ethereum state is public
by construction. Amounts and timing of deposits and releases are visible on the
public chain, and linking them to LEZ activity is a separate privacy problem,
handled by the consuming application through fixed denominations, delayed
submission, and anonymity-set growth (see [RFP-021](./RFP-021-wrapped-erc20.md),
Design Rationale, "The privacy requirement, stated precisely"). What this RFP
must guarantee is narrower and still essential: verifying and consuming an
attestation must not itself add a new correlation signal beyond what Ethereum
already exposes. A consumer that does its own privacy work correctly must not
have that work undone by the attestation path, which is why the verified
statement's public inputs are constrained in Functionality #8.

**It does not verify LEZ state on Ethereum.** The reverse direction, proving a
LEZ event natively on Ethereum, is specific to the bridge and stays in
[RFP-021](./RFP-021-wrapped-erc20.md).

### Consumption model

The primitive is delivered as a library callable from a consuming LEZ program,
not only as a standalone program with an account interface. A consumer must be
able to verify an attestation inline within its own transaction, so that
verification and the action it authorises are atomic and the consumer never has
to trust an intermediate account written by someone else.

A standalone program that verifies and records attestations to a public account
is a legitimate additional deployment shape, since it lets cost be amortised
when many consumers care about the same Ethereum state, and it is specified as a
soft requirement rather than a hard one. Where both shapes exist they must share
a single verification core, so the audit surface and the cost profile stay
common, following the pattern established in
[RFP-020](./RFP-020-redstone-oracle-adaptor.md).

### Cost is a primary deliverable

Whether sync-committee signature verification and Merkle-Patricia proof
verification fit inside a LEZ transaction budget, and at what cost, is the
central open question of this RFP. Signature verification inside RISC0 is known
to be expensive: prototype work on in-program secp256k1 ECDSA verification
([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench))
showed costs high enough to foreclose some designs on LEZ, and the sync
committee signs with BLS12-381 aggregate signatures, a different and heavier
primitive. Measuring and documenting the real cost, with a breakdown by
component, is therefore a first-class deliverable and not a side report, exactly
as it was for RFP-020. Consumers cannot size their own designs without those
numbers, and if the cost turns out to be unacceptable the measurement is the
input to a follow-on RFP proposing a BLS12-381 precompile for LEZ.

## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

01. Implement sync-committee tracking: given a trusted starting checkpoint,
    verify light-client updates and committee handoffs so the module can follow
    the canonical finalised chain forward across sync-committee periods without
    running a full node.
02. Verify that a supplied Ethereum block header is finalised, by checking
    sync-committee signatures over the light-client finality update, and reject
    any header that is not.
03. Verify Merkle-Patricia inclusion of a referenced piece of state under a
    verified finalised header: at minimum an account, a contract storage slot,
    and a receipt or log. Reject any inclusion proof that does not verify
    against the header's state or receipts root.
04. Evaluate a predicate over the verified state and emit a verified statement
    asserting that the predicate holds of finalised Ethereum state. The
    predicate interface must support at least equality and inequality
    comparisons over attested values, and must be expressible by a consuming
    program without modifying the module.
05. The verified statement binds the chain identity (network and chain ID) it
    refers to. A statement produced against one chain must be rejected by a
    verifier configured for another, tested across mainnet and at least one
    testnet with the same contract address on both.
06. The verified statement carries a deterministic, collision-resistant
    identifier derived from the attested Ethereum state, suitable for a consumer
    to key a nullifier on. Identical Ethereum state yields an identical
    identifier; distinct state yields a distinct one.
07. The module is callable as a library from a consuming LEZ program, so that
    verification and the action it authorises occur in a single transaction. A
    consumer must be able to integrate without depending on any account written
    by a third party.
08. Verifying and consuming an attestation must not introduce any correlation
    signal beyond what is already public on Ethereum. The public inputs of the
    verified statement must be limited to what the consumer explicitly chooses
    to expose, and must not reveal the identity of the consuming LEZ account or
    the consuming transaction. This must hold over full event and state diffs.
09. An admin authority (per [RFP-001](./RFP-001-admin-authority-lib.md),
    integrated via the [SPEL framework](https://github.com/logos-co/spel) where
    applicable) can configure the weak-subjectivity checkpoint, the supported
    chain and chain ID, and the finality parameters, per deployment. Document a
    procedure by which anyone can independently verify a configured checkpoint
    against public Ethereum sources before relying on the deployment.
10. Provide a checkpoint-refresh path for a deployment that has fallen outside
    the weak-subjectivity period, and document when a deployment is required to
    use it.
11. Proof generation must be a permissionless liveness role: no specific party
    may be required to produce an attestation, and any party declining to act
    must not block a user. Proposals must identify every off-chain participant
    the design requires and justify that none of them, individually or as a
    class, can block an attestation from eventually being produced.
12. The design must not assume Ethereum mainnet is the only target. The same
    module must be deployable, unmodified, against any Ethereum-consensus chain,
    mainnet or testnet, with the chain binding in Functionality #5 keeping
    deployments separate.

#### Usability

1. Build the attestation functionality for both consumers and admins in a Logos
   core module, so that different Logos ui modules can be built on it: producing
   an attestation, verifying one, and reading and administering the checkpoint
   and chain configuration.
2. Provide a CLI that covers producing an attestation for a given
   `(chain, contract, storage slot or log, predicate)` tuple, verifying one
   off-chain as a dry run, and reading and administering the configuration. The
   dry run must report the same typed error codes the on-chain path returns.
3. Any long-running off-chain component the design requires (for example a
   process that follows sync-committee handoffs and keeps update material
   available) must be provided as a **Logos module accompanied by a Logos Core
   headless CLI/daemon**, runnable standalone, supporting configurable Ethereum
   consensus and execution RPC endpoints, structured logging, and a clean
   shutdown path. Document the operator journey end-to-end: install, configure,
   run, monitor.
4. Provide an IDL for the LEZ-side module using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Return clear, actionable error messages for all failure modes: header not
   finalised, sync-committee signature invalid, committee handoff not verified,
   checkpoint stale or outside the weak-subjectivity period, inclusion proof
   invalid, predicate not satisfied, chain-ID mismatch, and malformed input.

#### Reliability

1. Verification is deterministic: the same inputs always yield the same verdict,
   independently reproducible from the test suite.
2. Verification is read-only with respect to consumer state: a rejected
   attestation consumes nothing and leaves the consumer able to retry.
3. A verified statement over a finalised header remains valid indefinitely.
   Later Ethereum activity, including subsequent sync-committee rotations, must
   never invalidate an attestation that was valid when produced.
4. Committee handoff tracking survives interruption: a component restarted after
   an outage resumes from its last verified state without re-verifying from the
   original checkpoint, and without accepting a handoff it has not verified.
5. Temporary RPC or connectivity failure on the Ethereum side leaves any
   off-chain component in a recoverable state, able to resume without
   duplicating work already done.
6. Proposals must integrate mature, audited implementations of BLS12-381
   signature verification, Merkle-Patricia proof verification, and the
   light-client protocol rather than reimplementing cryptographic primitives
   from scratch.
7. CI must be green on the default branch.

#### Performance

1. Verifying an attestation and evaluating its predicate must complete within a
   single LEZ transaction at the per-transaction compute budget in force at
   delivery time, in the library consumption path of Functionality #7. If this
   proves infeasible at delivery time, the measurement in Performance #2 stands
   as the deliverable and the shortfall must be documented with the specific
   component responsible.
2. Cost measurement is a primary deliverable, not a side report. Measure and
   document, with a breakdown by component: BLS12-381 aggregate signature
   verification over a sync-committee update, committee handoff verification,
   Merkle-Patricia inclusion proof verification for an account, a storage slot
   and a log, and predicate evaluation. Report compute units, proof time, and
   proof size for each, extending the measurement methodology from
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md). Numbers must be reproducible
   from the test suite.
3. Document the amortised per-attestation cost of sync-committee tracking at the
   recommended operating cadence, separately from the per-attestation
   verification cost, since the two amortise differently across consumers.
4. Any proving the user's own device must perform has to be practical on the
   desktop hardware Basecamp runs on. Measure and document wall-clock time and
   peak memory on a mid-range laptop and on the lowest specification the team
   declares as supported, and state that minimum explicitly.
5. Document end-to-end attestation latency, from the Ethereum event to a
   verified statement usable on LEZ, broken down by finality wait, update
   availability, proof generation, and on-chain verification.
6. Document the compute resources (CPU, RAM, time) required to run any off-chain
   component the design requires.
7. Document the growth rate and on-chain storage cost of any module state that
   accumulates with usage, in particular committee-tracking state, with
   projections over one and five years of continuous operation.
8. Document the cost delta between the in-program BLS12-381 path and a
   hypothetical native precompile, so the measurement can inform whether a
   follow-on precompile RFP is warranted.

#### Supportability

01. The module is deployed and tested on LEZ testnet, attesting to state on a
    public Ethereum testnet.
02. End-to-end integration tests exercise attestation production and consumption
    against a LEZ sequencer (standalone mode) and an Ethereum test network or
    local fork, and are included in CI.
03. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
04. A README documents end-to-end usage: module addresses, deployment steps,
    configuration of the checkpoint and chain, and step-by-step instructions for
    producing and consuming an attestation via the CLI.
05. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the core module, covering the developer integration journey for a
    consuming LEZ program.
06. The consumer documentation must include a **"Recommended Consumer Pattern"**
    section covering: enforcing uniqueness with a nullifier keyed on the
    statement identifier from Functionality #6, verifying the chain binding
    matches the consumer's expectation, handling each typed error code, and the
    recommended behaviour when an attestation cannot be produced (refuse the
    action, never fall back to an unverified input).
07. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the CLI and any operator-facing components, covering the core user and
    operator journeys respectively.
08. Provide a **trust assumptions document** covering: the honest sync-committee
    supermajority assumption and the cost of the attack it admits; the
    weak-subjectivity checkpoint, who supplies it and how a verifier confirms it
    independently; what an attestation does and does not prove; what every
    off-chain participant in the design can observe; and the conditions under
    which the guarantees degrade or fail.
09. The module undergoes an independent third-party security audit of the
    consensus and inclusion verification logic before any mainnet-facing
    deployment; the audit report must be published.
10. The deliverable must be published on the module catalog.
11. The repository must use the standard Logos GitHub Actions.

#### + Verification Security

1. Verification must reject a fabricated chain: given a self-consistent header
   and inclusion proof over state that was never finalised on the canonical
   chain, verification must fail. This is the central adversarial test of the
   RFP and must be exercised explicitly, not implied by other tests.
2. Verification must reject: a header signed by a committee that was never
   validly handed off from the configured checkpoint; a sync-committee signature
   below the required participation threshold; a valid inclusion proof against a
   header that is not finalised; a valid finalised header with a tampered state
   or receipts root; a proof replayed against a different chain ID; and a
   predicate asserted over state the inclusion proof does not cover.
3. The verifier must be deployed as an immutable program with an explicit
   migration path (deploy a new version, redirect consumers to it) in preference
   to an upgradeable one governed by a mutable key. Legitimate, authorised
   upgrades have shipped catastrophic verification bugs in production bridges
   (see
   [Appendix: Bridges and Wrapped Tokens](../appendix/bridges-and-wrapped-tokens.md)),
   which an immutable program forecloses by construction. Document the migration
   mechanism and how consumers are expected to move across a migration.
4. Document the failure modes that follow from a compromised or colluding sync
   committee, and what a consuming application can do to bound its exposure (for
   example, caps and freeze authorities on the consumer side, as
   [RFP-021](./RFP-021-wrapped-erc20.md) specifies).
5. User-facing and developer-facing documentation must state the trustless
   verification model and the liveness-only role of any off-chain participant
   (see Design Rationale, "The anchor: Ethereum finality via the sync
   committee").

### Soft Requirements

1. **Standalone attestation program.** In addition to the library path
   (Functionality #7), provide a LEZ program that verifies attestations and
   records verified statements to a public account, so cost can be amortised
   when many consumers care about the same Ethereum state. It must share a
   single verification core with the library path, following the pattern in
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md), so that the audit surface
   and the cost profile stay common.

2. **Batching.** Amortise verification cost across multiple attestations over
   the same finalised header in a single transaction, analogous to the
   multi-feed batching soft requirement in RFP-020.

3. **Historical state.** Attest to state under a header older than the
   currently-tracked finalised head, for example via historical summaries in the
   beacon state, so a consumer can prove a fact about Ethereum's past rather
   than only its recent finalised present.

4. **Richer predicates.** Support predicates over multiple attested values in a
   single statement (for example, an account balance and a storage slot under
   the same header), so a consumer can express a compound condition without
   producing and correlating several attestations.

5. **Pluggable proof components.** Design the proof-system components so that
   future zkVM improvements, proof compression, or hardware acceleration can be
   adopted without restructuring the module or its consumer interface.

6. **Additional chains.** Extend the attestation to other chains with a
   light-client-verifiable consensus, each served by its own configuration per
   Functionality #12.

### Out of Scope

The following are explicitly excluded from this RFP:

- **Verifying LEZ state or events on Ethereum.** The reverse direction is
  specific to the bridge and stays in [RFP-021](./RFP-021-wrapped-erc20.md).
- **Any application action.** Minting, escrow release, settlement, and every
  other action belong to the consuming program. This RFP emits a verified
  statement and stops there.
- **Uniqueness enforcement.** The primitive supplies the statement identifier
  (Functionality #6) and the guidance (Supportability #6); the nullifier or
  spent-marker itself belongs to the consumer (see Design Rationale, "Uniqueness
  is the consumer's job, not the primitive's").
- **Hiding Ethereum-side activity.** Ethereum state is public by construction.
  Unlinkability between Ethereum activity and LEZ activity is the consuming
  application's problem; this RFP's obligation is only to avoid adding a new
  correlation signal (Functionality #8).
- **Adding a BLS12-381 precompile to LEZ.** The in-program path is the
  deliverable here. A precompile becomes a candidate for a follow-on RFP if and
  only if the cost measurement in Performance #2 shows the in-program path is
  too expensive for production use.
- **Circuit optimisation or custom zkVM accelerators.** LEZ runs on RISC0, so
  proposals should leverage mature existing implementations rather than
  implementing novel circuits or optimisation techniques.

## ⚠ Platform Dependencies

### Hard dependencies

#### Admin authority (RFP-001)

The Functionality requirements specify that an admin authority configures the
weak-subjectivity checkpoint, the supported chain and chain ID, and the finality
parameters. These admin-gated functions require the standardised admin authority
library from [RFP-001](./RFP-001-admin-authority-lib.md).

#### Event emission (LP-0012)

Structured on-chain events let clients and off-chain components follow committee
handoffs and attestation submissions without polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
is **closed** (delivered).

#### RISC0 zkVM

The module verifies Ethereum consensus and state in-program on LEZ. Because LEZ
itself runs on RISC0, a production-ready zkVM, this is a LEZ-runtime dependency
rather than a choice the proposal makes; proposals must leverage mature RISC0
implementations (e.g. [Zisk](https://github.com/risc0/zisk)) for LEZ-side
proving rather than building custom circuits.

#### Logos Ethereum core module

Any Ethereum-side interaction, including fetching headers, update material, and
inclusion proofs, must use the Logos Ethereum core module, including its
verified proxy features.

## 👤 Recommended Team Profile

Team experienced with:

- Ethereum consensus and the light-client protocol (sync committees, finality
  updates, committee handoffs, weak subjectivity)
- BLS12-381 signature verification and aggregate signature schemes
- Merkle-Patricia trie proof verification against Ethereum state and receipts
  roots
- Zero-knowledge proof systems and the RISC0 zkVM (guest program development,
  proof generation and verification, public/private input handling, and cost
  characterisation, since cost measurement is a primary deliverable)
- LEZ program development and on-chain proof verification
- Security auditing of consensus verification logic (fabricated-chain attacks,
  replay, committee-handoff attacks, checkpoint staleness)
- Designing libraries consumed by other protocol teams, including interface
  stability and integration documentation

## ⏱ Timeline Expectations

Estimated software delivery duration: **8–12 weeks**. The scope is a single
primitive with no application logic, but sync-committee tracking, in-zkVM
BLS12-381 verification, and the cost measurement deliverable are each
substantial. This excludes the third-party audit lead time required before any
mainnet-facing deployment (Supportability #9), which is typically procured and
scheduled separately.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md)
- [RFP-003 — Atomic Swaps](./RFP-003-atomic-swaps.md) (trustless swap path for
  chains without general smart-contract expressiveness)
- [RFP-020 — RedStone Off-Chain Oracle Adaptor for LEZ](./RFP-020-redstone-oracle-adaptor.md)
  (reference for in-program verification cost measurement, and for the shared
  verification core between a library and a program deployment shape)
- [RFP-021 — Privacy-Preserving Wrapped ERC-20 and Ether Bridge for LEZ](./RFP-021-wrapped-erc20.md)
  (primary consumer; the mint side depends on this attestation)
- [RFP-023 — Native Gas Token Bridge for LEZ](./RFP-023-gas-token-bridge.md)
  (consumer; the gas-token release path depends on this attestation)
- [Appendix: Bridges and Wrapped Tokens](../appendix/bridges-and-wrapped-tokens.md)
  (bridge failure taxonomy, including verification-logic bugs)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
- [Ethereum Altair light-client specification](https://github.com/ethereum/consensus-specs/tree/dev/specs/altair/light-client)
- [Ethereum weak subjectivity specification](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/weak-subjectivity.md)
- [RISC0 — Zero-Knowledge VM](https://github.com/risc0/risc0)
- [Zisk — RISC0 Proof Generation](https://github.com/risc0/zisk)
- [`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench)
  (prototype measurements of in-program signature verification cost on RISC0)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
