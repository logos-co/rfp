<!-- Draft staged in logos-co/rfp; will move to logos-co/lambda-prize once accepted. -->

<!-- Don't forget to add/update this prize in the table in README.md when moved. -->

# LP-0019: Off-Chain Verifiable Reputation for Atomic-Swap Makers [Draft]

**`Status`**:

- Draft: Not yet ready
- Open: Ready for application
- Completed: Submission accepted, prize completed

**`Logos Circle: N/A`**

## Overview

In any atomic-swap protocol there are abandonment paths the on-chain settlement
contract cannot adjudicate. The clearest example sits in the LEZ↔XMR direction
(RFP-003 LEZ atomic-swap protocol's Tier-2 case): a maker who has received a
counterparty's Monero lock can refuse to advance on LEZ, and the LEZ contract
cannot tell whether the maker is malicious or the counterparty never actually
locked. The Monero side is not observable from LEZ without view-key disclosure
(which would deanonymise the swap output). The protocol falls back to a
he-said-she-said dispute; the only recourse is *reputation*.

The Logos cross-chain DEX bundle treats this as a real gap in the vanilla
atomic-swap protocol and does not pretend slashing or attestor-based mechanisms
can close it without reintroducing trust. This prize is for a **mechanism that
lets honest counterparties publish third-party-verifiable evidence of maker
misbehaviour**, so a reputation system built on top of vanilla atomic swaps
(RFP-003) can be both privacy-respecting and adversarially robust. The Logos
team does not pre-judge how. Solvers may use view-key disclosure with privacy
mitigations, FCMP++-grade zk proofs, multi-party attestation schemes, watchtower
designs, or any combination, as long as the resulting reputation system survives
the evaluation against the success criteria below.

## Motivation

A reputation system that records only successful swaps is useless: it cannot
deter misbehaviour because misbehaviour leaves no on-chain trace. A reputation
system that records every counterparty complaint without filtering is worse than
useless: a malicious *taker* can manufacture false complaints against an honest
maker.

The hard problem is producing evidence of "maker received a counterparty lock
and did not advance" that **any third party with access to the relevant
blockchain state can verify, without trusting the complainant**. This is
non-trivial in the LEZ↔XMR case because the Monero output is not directly
readable from outside the bilateral context, and the natural disclosure
(view-key + lock-amount) deanonymises the swap.

A competitive prize is the right mechanism because the design space is large and
no published solution exists:

- **Naive view-key disclosure**: complainant publishes the shared view key plus
  the lock transaction so any third party can verify the lock against the
  maker's signed quote. Cost: the swap output is deanonymised forever to anyone
  holding the proof bundle.
- **Selective-disclosure zk proofs over Monero output structure**: prove "an
  output of the quoted amount exists at the stealth address derived from this
  quote's joint-key transcript" without exposing the view key. Cost: requires
  either FCMP++-grade primitives that are pre-production, or a custom proof
  system over Monero's CLSAG ring-signatures.
- **Multi-party attestation**: a small committee of attestors observes both
  chains and signs off on disputed cases. Cost: reintroduces a trust layer the
  bundle is trying to avoid.
- **Watchtower designs**: paid third parties run both Monero and LEZ nodes and
  earn fees for issuing attested verdicts. Cost: the watchtower itself becomes a
  target.
- **Hybrid**: tiered reputation where most claims are unverifiable but the
  highest-stakes claims demand a verifiable proof bundle.

Each carries trade-offs in privacy cost, cryptographic complexity, deployability
today, and adversarial robustness. The Logos team does not want to pre-judge the
answer; this prize is open to any approach.

The prize complements but does not depend on LP-0018 (atomic-swap anti-spam
mechanism): a robust reputation system reduces the need for protocol-level
deterrents by making repeated misbehaviour visible across swaps; LP-0018's
mechanisms deter individual misbehaviour without needing cross-swap history.
Both can coexist.

## Success Criteria

### Functionality

- [ ] An honest counterparty (taker or maker) can publish a proof bundle for a
  failed swap that a third party with access to the relevant blockchain state
  (LEZ + the relevant external chain) can verify *without trusting the
  complainant*. The verification produces one of: valid complaint, invalid
  complaint, indeterminate (with documented reason).
- [ ] Covers at least the LEZ↔BTC and LEZ↔ETH cases. Bonus: covers LEZ↔XMR with
  documented trade-offs (privacy cost of disclosure, dependency on future Monero
  primitives if relevant).
- [ ] The mechanism distinguishes:
  - The complainant fabricating a quote: caught by signature verification on the
    quote.
  - The complainant fabricating a lock: caught by the verifier checking the
    external-chain lock against the quote's derived stealth-address / script /
    contract.
  - The accused successfully rebutting (e.g. presenting their own LEZ activity
    proving they advanced): caught by the verifier checking LEZ state.
- [ ] **Sybil resistance.** The cost of building a high-reputation identity to
  then defect must exceed the expected gain from defecting. Sybil identities
  should not be able to manufacture clean reputation cheaply.
- [ ] **Spam resistance.** A malicious taker cannot mass-publish fake complaints
  to deny-of-service the reputation system. Either the cost of publishing a
  complaint is bounded below (e.g. micro-fee) or the verifier filters cheaply.
- [ ] **Aggregation method documented.** A maker's overall reputation score is a
  function of (verified positive completions, verified valid complaints,
  possibly weighted by complainant reputation). The submission must specify the
  function and justify it.
- [ ] **Privacy on the complainant side.** A taker complaining about a maker
  should not be forced to reveal which other swaps that taker has done, unless
  they choose to. (The accused maker is necessarily identifiable since the
  complaint is about their identity.)
- [ ] **Existing-project comparison.** The submission must identify at least one
  existing reputation system (e.g. Wormhole Guardian set behaviour reporting,
  Thorchain bond-to-pooled slashing rules, decentralised review aggregators,
  EigenLayer AVS slashing, or other) and explain how the chosen design relates
  to or differs from it.

### Usability

- [ ] Provide a module/SDK that can be used to build Logos modules for
  interacting with the reputation system (querying maker reputation, publishing
  complaints, generating proof bundles).
- [ ] Provide a Logos Basecamp app GUI with local build instructions,
  downloadable assets, and loadable in Logos app (Basecamp).
- [ ] Provide an IDL for any LEZ programs introduced, using the
  [SPEL framework](https://github.com/logos-co/spel).
- [ ] Surface the maker's reputation in a way takers can inspect *before*
  initiating a swap.

### Reliability

- [ ] Disputed claims (where the verifier returns "indeterminate") are handled
  with a documented policy: they are recorded as such, not silently dropped, and
  do not falsely degrade either party's reputation.
- [ ] The reputation database is censorship-resistant: it does not depend on a
  single host or central index.
- [ ] Storage primitives for the proof bundles (where they live, how long they
  persist) are specified. Logos Delivery is the expected substrate.

### Performance

- [ ] Document the storage and bandwidth cost per complaint, per verification,
  and per reputation query.
- [ ] State the latency from publishing a complaint to it being aggregated into
  the maker's reputation score.
- [ ] Document compute unit (CU) cost of any on-chain LEZ operations introduced.

### Supportability

- [ ] Any LEZ programs introduced are deployed and tested on LEZ devnet/testnet.
- [ ] End-to-end integration tests run against a LEZ sequencer (standalone mode)
  and are included in CI.
- [ ] CI must be green on the default branch.
- [ ] A README documents end-to-end usage: how to publish a complaint, how to
  query reputation, how the verifier works.
- [ ] A reproducible end-to-end demo script is provided and works against a real
  local sequencer with `RISC0_DEV_MODE=0`.
- [ ] A recorded video demo of the end-to-end flow is included in the
  submission; the recording must show terminal output (including proof
  generation) to confirm `RISC0_DEV_MODE=0` was active.
- [ ] The demo includes at least one "valid complaint" scenario and one
  "fabricated complaint rejected" scenario.

## Scope

### In Scope

- The reputation mechanism: data model, proof-bundle structure, verifier design,
  aggregation function, anti-spam and anti-sybil mechanisms.
- Storage and distribution: where proof bundles live (Logos Delivery is the
  expected substrate); how reputation scores are computed and propagated.
- Client-side: SDK for publishing complaints, generating proof bundles, and
  querying reputation.
- A reference integration with RFP-003 LEZ atomic-swap maker daemon, where the
  maker's reputation is auto-updated on completed and failed swaps.
- Documentation: cryptographic approach, privacy guarantees, adversarial model,
  comparison against at least one existing reputation system in the wild.

### Out of Scope

- Changing the underlying RFP-003 atomic-swap cryptography.
- Building a protocol-level slashing mechanism on top of the reputation (that
  would be the role of LP-0018 (anti-spam mechanism); this prize stops at
  producing the verifiable evidence and reputation score).
- A polished consumer UI beyond what's needed for the demo.
- LEZ↔XMR coverage at FCMP++-grade privacy. A submission that addresses LEZ↔BTC
  and LEZ↔ETH cleanly is sufficient; LEZ↔XMR with current Monero primitives
  (potentially involving view-key disclosure) is a bonus.

## Prize Structure

- **Total Prize:** $X
- **Effort:** L

> Leave prize pool blank — this will be determined by the Logos team. Single
> winner by default.

## Eligibility

Open to any individual or team. Submissions must be original work. Teams must
hold the rights to all submitted code and agree to license it under MIT or
Apache-2.0.

## Submission Requirements

A submission must include:

- A public repository containing the LEZ program(s) (if any), client SDK,
  proof-bundle format spec, verifier implementation, and aggregation logic.
- A written design document covering the data model, cryptographic primitives,
  privacy analysis, adversarial model, and comparison against at least one
  existing reputation system.
- A narrated video walkthrough demo showing (a) honest swap completion with
  reputation update, (b) a valid complaint scenario where the verifier accepts
  the proof bundle, (c) a fabricated complaint scenario where the verifier
  rejects it, and (d) the maker reputation score visible to a prospective taker.
  The demo must show terminal output including any proof generation with
  `RISC0_DEV_MODE=0`.
- A FURPS self-assessment (see
  [solution template](https://github.com/logos-co/lambda-prize/blob/main/solutions/LP-0000.md)).

## Evaluation Process

By default, submissions are evaluated first-come-first-served against the
success criteria. The first submission that meets all criteria wins.

Evaluators will independently clone the repository and run the demo script from
a clean environment; the script must succeed without modification. Evaluators
will exercise the valid-complaint and fabricated-complaint scenarios themselves.

Because the design space is large and multiple valid approaches exist,
evaluators may rank tied submissions on:

1. Adversarial robustness: how many distinct attack vectors (sybil, spam,
   false-complaint, false-rebuttal) the mechanism handles with explicit
   countermeasures.
2. Privacy cost: lower disclosure to third parties is preferred.
3. Deployability today: solutions that work with currently-available Monero
   primitives (where applicable) rank above solutions that depend on
   pre-production cryptography.
4. Integration cleanliness with RFP-003's existing maker daemon.

The following policies apply to all prizes (see
[evaluation policies](https://github.com/logos-co/lambda-prize/blob/main/README.md#evaluation-policies)):

- **Submissions:** each builder (or team) is allowed a maximum of **3
  submissions** per prize, with at most **one submission/review per week**.
- **Feedback:** initial evaluation feedback is limited to a pass/fail indication
  against the success criteria.

## Resources

- [RFP-003: Atomic Swaps with LEZ](https://github.com/logos-co/rfp/blob/master/RFPs/RFP-003-atomic-swaps.md)
  — the underlying atomic-swap protocol whose makers this reputation system
  tracks.
- [LP-0018: Anti-Spam Mechanism for Atomic Swaps](./LP-0018-atomic-swap-anti-spam.md)
  — complementary prize on protocol-level deterrence. Both can be deployed
  together.
- [appendix/atomic-swaps-primer.md](https://github.com/logos-co/rfp/blob/master/appendix/atomic-swaps-primer.md)
  — atomic-swap mechanics; relevant for understanding what events the reputation
  system records.
- [appendix/cross-chain-trust-model-contrast.md](https://github.com/logos-co/rfp/blob/master/appendix/cross-chain-trust-model-contrast.md)
  — surveys existing reputation-adjacent mechanisms in deployed protocols
  (Wormhole Guardian set, Thorchain bonded validators).
- [Monero, Zero to Monero 2.0 §Payment Proofs](https://www.getmonero.org/library/Zero-to-Monero-2-0-0.pdf)
  — `check_tx_proof` family (OutProofV2, InProofV2) for understanding what
  bilateral disclosure looks like on Monero.
- [Monero, FCMP++ announcement (2024-04-27)](https://www.getmonero.org/2024/04/27/fcmps.html)
  — research direction for non-disclosing Monero proofs. Solvers targeting
  LEZ↔XMR may want to design with FCMP++ readiness in mind.
- [Wormhole Guardian set](https://wormhole.com/docs/protocol/infrastructure/guardians/)
  — example of a reputation-substituting-for-bond mechanism (Proof-of-Authority
  committee of named entities).
- [Thorchain RUNE bond-to-pooled docs](https://docs.thorchain.org/understanding-thorchain/rune)
  — example of a slashable reputation mechanism in a federated-signer DEX (the
  contrast point against atomic-swap-only protocols).

## Potential for Subsequent λPrizes

If this prize is awarded for a LEZ↔BTC and LEZ↔ETH mechanism, a follow-up prize
may be opened for LEZ↔XMR coverage once non-disclosing Monero proof primitives
reach production-ready status (FCMP++ is the leading candidate as of 2026-05).
