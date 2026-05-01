---
id: RFP-020
title: "RedStone Off-Chain Oracle Adaptor for LEZ"
tier: M
funding: $XXXXX
status: open
dependencies: See Platform Dependencies section
category: Developer Tooling & Infrastructure
---


# RFP-020 — RedStone Off-Chain Oracle Adaptor for LEZ

## 🧭 Overview

Build a RedStone off-chain oracle adaptor for LEZ: a public-mode
LEZ program that verifies RedStone-signed data packages, exposes
the resulting prices through the canonical oracle price account
standard defined in [RFP-019](./RFP-019-twap-oracle.md), and
supports day-one delivery of XMR/USD and ZEC/USD feeds. RedStone's
data packages are signed with secp256k1 + keccak256 by its data
nodes; verification on LEZ runs as in-program code inside the
RISC-V zkVM (no cross-chain bridge, no Wormhole dependency). The
adaptor uses a push-mode aggregator pattern: a public-mode program
verifies signatures on the write side, stores the result in a
public price account, and consumers (including private-execution
programs) read the slot. This RFP is scoped to the RedStone
adaptor only; on-chain TWAP is in RFP-019, and a Pyth adaptor
(which adds a Wormhole dependency) is deferred to a future RFP.

LEZ is RISC0-based, so any signature scheme can be implemented in
program code. The open question is whether the resulting program
cost is acceptable. This RFP makes that question its first
deliverable: implement signature verification in RISC-V, measure
the cost, document the result. If the measured cost is acceptable
for the push-mode aggregator's update cadence, the adaptor ships
on the existing runtime. If it is not, the measurement becomes the
input to a follow-on RFP that proposes adding a secp256k1 ECDSA +
keccak256 precompile to LEZ for the public-mode write side. The
precompile is therefore an optimisation path, not a precondition
for this RFP.

(Scope note: this RFP is about asset-price oracles for DeFi
applications. It is unrelated to the RLN service-attestation oracle
work on the anon-comms roadmap.)

## 🔥 Why This Matters

Logos's thesis is private DeFi: assets, applications, and users
that the broader web3 stack does not yet serve well. Privacy
collateral, in particular Monero (XMR) and Zcash (ZEC), is the
day-one asset class that distinguishes LEZ from a generic L2 or
appchain DeFi deployment. The LSC stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), the
privacy-preserving DEX
([RFP-004](./RFP-004-privacy-preserving-dex.md)), wrapped privacy
assets, and other cross-chain primitives all need a USD reference
price for XMR and ZEC to function. Without one, none of those
applications can ship.

Across the surveyed off-chain oracle providers, RedStone is the
only one that combines: support for both XMR and ZEC in its public
token registry, a portable connector pattern (single secp256k1
ECDSA + keccak256 verification path that works the same on every
host chain), no cross-chain bridge requirement, and a self-serve
deployment path that does not require an oracle-team business
engagement. Pyth covers both feeds and adds higher publisher
counts and confidence intervals, but is gated on Wormhole
integration on LEZ; it should land as a fast-follow in a future
RFP. Chainlink is permissioned and not self-serve. DIA Lumina is
permissionless but requires bespoke per-chain deployment. See
[Appendix: Oracle Ecosystem, Privacy-Asset Feed Availability](../appendix/oracle-ecosystem.md)
for the full coverage matrix.

The combination of "private DeFi needs XMR and ZEC" and "RedStone
is the only path that is self-serve on LEZ today" makes this the
priority off-chain oracle integration for LEZ.

### Production security is a business decision

This RFP delivers a swappable building block, not a production
stablecoin oracle. The LSC stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)) faces a
genuine choice for production:

- **Path A (LSC/USD direct).** Use external oracles
  (RedStone, Pyth) for LSC/USD. Pros: single source, simpler
  integration. Cons: LSC/USD off-chain liquidity will be thin
  early; volatile markets create a manipulation surface; the real
  problem is low CEX liquidity, which a stable AMM does not fully
  fix.
- **Path B (LGS/USD + LGS/LSC composite).** Use an external
  LGS/USD feed combined with an on-chain LGS/LSC TWAP. Pros: LGS
  is expected to have deeper CEX liquidity than LSC once the
  network reaches scale; external oracles for LGS/USD are
  battle-tested. Cons: the LGS/LSC TWAP becomes the manipulation
  bottleneck, which is exactly the low-liquidity vulnerability the
  TWAP RFP raises.

This RFP does not pick A or B. It provides the off-chain oracle
half of either path: USD prices for LSC, LGS, XMR, ZEC, and any
other asset RedStone supports. RFP-019 provides the TWAP half
needed for Path B. Whichever path the LSC implementer chooses for
production, this adaptor remains the swap-in for the off-chain
component.

## 🏗 Design Rationale

### Public-mode aggregator with private-account composability

The adaptor runs as a public-mode LEZ program with no confidential
state. Signature verification, data-package decoding, and price
publication are all visible to any caller. Any LEZ dapp can read
the same canonical price.

This shape is determined by where signature verification can run
on LEZ. LEZ is a RISC-V zkVM built on RISC0; any code that runs
inside a private transaction has to be expressible inside the
RISC-V zkVM circuit, so a private transaction that wants to verify
a secp256k1 ECDSA signature has two options, both unappealing:
verify the signature inside the privacy circuit (forfeits the
batching benefits that make ZK proof amortisation work; RISC0
elliptic-curve performance for this primitive is currently
unmeasured), or place the signature in the transaction journal
where it is publicly disclosed (breaks the privacy of the
transaction). Neither option preserves both efficiency and
privacy.

The adaptor therefore runs the verifier in a public-mode
aggregator: signatures are recovered once per update on the write
side, and the verified price plus timestamp are stored in a public
price account. Private-execution programs compose with the price
by reading the public account, not by carrying signed payloads
inline. Cost is paid once per update and amortises across all
downstream reads, public and private. Confidential execution is
reserved for application-layer protocols that consume oracle
prices (for example, private DEX swaps in
[RFP-004](./RFP-004-privacy-preserving-dex.md)); the price feed
itself stays public.

Pull-mode reads (where a public consumer transaction carries a
signed payload and verifies inline) remain technically possible on
LEZ inside public execution, but are out of scope for this RFP
because they don't extend to private execution and because the
push-mode aggregator gives strictly better cost amortisation for
the LEZ DeFi consumer set. They can be revisited in a follow-on
once measured cost data is in.

A LEZ-specific freshness pattern follows from the public / private
execution split. A user who needs a price fresher than the
heartbeat's last update can submit a public transaction that
pushes a fresh signed payload to the aggregator account, then
submit a private transaction immediately after that reads the
just-updated public price. Verification cost is paid in the public
path (cheaper, especially under the precompile follow-on); the
private transaction does no signature work. This recovers pull
mode's "fresh at transaction time" property for private consumers
without paying the in-circuit cost in the privacy proof. The
adaptor program already accommodates this: any caller can submit
a valid signed payload and the program writes if signatures and
timestamps check out.

The same mechanism enables a **consumer-pays push variant** that
does not require a dedicated relayer at all. Users push when they
need a fresh price; idle periods incur zero update cost; the
aggregator only advances when someone needs it. This is
operationally pull (consumer-pays, on-demand) but structurally
push (the program owns the public price account that downstream
private consumers read from). Whether to run a heartbeat relayer
in addition (RedStone's own pusher, a sovereign relayer, or
neither) is a deployment-time choice: a heartbeat keeps the slot
warm for read-only consumers; consumer-pays push keeps the cost
model strictly proportional to demand. Both can coexist; the
program logic does not distinguish between them.

### RISC-V verification path and the precompile question

This RFP implements signature verification in RISC-V program code,
running inside RISC0. There is no host primitive to call: the
recovery is an in-program ECDSA + keccak256 path written against
existing Rust crates (k256 / sha3 / equivalents) and proved by
RISC0 along with the rest of the program.

This is the central technical bet of the RFP. RISC0 elliptic-curve
performance for secp256k1 ECDSA recovery and keccak256 hashing has
not been comprehensively measured in the LEZ runtime; the LEZ team
has discussed testing but deprioritised it. The first concrete
deliverable of this RFP is therefore the measurement: implement
the verifier in RISC-V, run it on LEZ, document the cost
(compute units / proof time / proof size / per-update bytes) for
both the per-signature recovery and the full 3-of-N aggregator
write.

Two outcomes are possible from that measurement:

1. **Measured cost is acceptable for the push-mode aggregator.**
   The adaptor ships on the runtime as it stands. The aggregator's
   update cadence amortises the per-update cost across all
   downstream reads. No runtime change required.
2. **Measured cost is unacceptable.** The measurement becomes the
   input to a follow-on RFP that proposes adding a secp256k1 ECDSA
   + keccak256 precompile to LEZ for use by public-execution
   programs. A precompile lives outside the ZK proof boundary and
   is invoked as native validator code, so the cost goes from
   "ZK-proven elliptic-curve operations" to "native ECDSA recovery
   + keccak", which is the cost profile RedStone's existing
   connectors assume on every other chain. The precompile is an
   optimisation path conditional on the measurement, not a
   precondition for this RFP.

The applicant should therefore design the verification path so
that swapping in a precompile in a later release is a localised
change (a single trait implementation or syscall wrapper), not a
restructuring of the program.

### Why RedStone first

Three reasons specific to LEZ's constraints:

1. **Privacy-asset coverage with no bridge.** Both XMR and ZEC are
   in RedStone's public token registry, and the RedStone connector
   pattern is fully self-serve: deployment does not require an
   oracle-team engagement, a bridge, or a per-chain registration
   step. LEZ can deploy and exercise these feeds without touching
   any external infrastructure. Pyth and DIA both cover the same
   assets but require either Wormhole (Pyth) or bespoke per-chain
   deployment (DIA Lumina) before they work on a new chain.
2. **Single verification primitive, no bridge.** RedStone's
   signature scheme is plain m-of-N secp256k1 ECDSA over keccak256
   (typically 3-of-N). Verification on LEZ is in-program ECDSA
   recovery and keccak256 hashing inside RISC0; the cost profile
   is the open variable this RFP measures (see "RISC-V verification
   path and the precompile question"). Pyth's full 13-of-19
   Wormhole VAA verification is heavier in two ways: it adds a
   Merkle proof on top of more signatures, and it presupposes a
   Wormhole guardian-set tracking program on LEZ that does not yet
   exist. RedStone has neither cost.
3. **Independent of LEZ's external integration timeline.**
   Choosing RedStone first decouples the oracle layer from when
   Wormhole on LEZ is decided. Pyth then fast-follows in a future
   RFP, contributing higher publisher counts (especially the
   roughly 80+ on XMR/USD versus RedStone's smaller per-feed
   roster) and confidence intervals that RedStone does not
   natively expose.

See
[Appendix: Oracle Ecosystem, Signature Verification Schemes](../appendix/oracle-ecosystem.md)
for the full per-scheme analysis and citations.

### Conformance to the canonical price account standard

The canonical oracle price account standard is owned by RFP-019
(see "LEZ oracle data standard" in that RFP's Design Rationale).
The RedStone adaptor populates the same struct as the on-chain
TWAP source: price, timestamp, source identifier, confidence
interval (zero, since RedStone does not publish one), and
circuit-breaker dispute flag. Consuming protocols query a single
data layout and remain agnostic to whether the price came from
TWAP, RedStone, or any future provider.

If RFP-019 has not yet shipped the canonical struct when this RFP
is delivered, the team must define a forward-compatible minimal
struct using append-friendly account-data conventions, so that a
later RFP-019 release can extend the struct without breaking
consumers.

### Pull-model fee structure

Proposals must specify a fee model covering: who pays for oracle
updates (consumer, protocol, or subsidised), when fees are charged
(per query, per update, per registration), the fee rate or
formula, and where fees are routed (protocol treasury or burned).
Because RedStone is a pull oracle, the cost model naturally
follows the consumer that submits the signed data packages; the
adaptor does not need to fund a dedicated node operator pool. The
fee model should be sustainable without ongoing subsidies once LEZ
reaches moderate TVL.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a public-mode LEZ program (push-mode aggregator) that
   accepts signed RedStone data packages, recovers each signer's
   public key via in-program secp256k1 ECDSA recovery (with
   keccak256 hashing) running inside the RISC-V zkVM, and verifies
   that the recovered public keys match the configured set of
   authorised RedStone data nodes for the requested feed.
   Structure the verification path so that swapping the in-program
   recovery for a future host primitive (precompile or syscall) is
   a localised change.
2. Verify the M-of-N signer threshold for each feed (configurable
   at registration; default 3-of-N consistent with RedStone's
   reference parameters) and reject any data package that does
   not meet the threshold.
3. Decode the RedStone data package format (asset identifier,
   value, timestamp, signer set) and reject any package whose
   timestamp is older than a configurable `maxAge`, whose value is
   zero, negative, or otherwise invalid, or whose asset identifier
   does not match the registered feed.
4. Publish the verified price into a canonical oracle price
   account conforming to the standard defined in RFP-019. The
   adaptor must populate price, timestamp, source identifier (a
   constant identifying RedStone), confidence interval (zero), and
   circuit-breaker dispute flag (always cleared at write time;
   the TWAP program in RFP-019 owns dispute-state transitions).
5. The adaptor program owner can register new RedStone feeds (by
   asset identifier, M-of-N threshold, and authorised signer set),
   update an existing feed's signer set on RedStone roster
   changes, and deregister feeds.
6. XMR/USD and ZEC/USD feeds must be registered and exercised on
   LEZ devnet/testnet as part of the deliverable.

#### Usability

1. Provide an SDK that can be used to build Logos modules for
   submitting RedStone data packages and reading verified prices
   from the canonical price account.
2. Provide a Logos mini-app GUI (off-chain feed dashboard) with
   local build instructions, downloadable assets, and loadable in
   Logos app (Basecamp) via git repo. The dashboard must display:
   live prices for each registered feed, the configured signer
   set, the current M-of-N threshold, the latest data-package
   timestamp, and the staleness of each feed.
3. Provide a CLI that covers core functionality: submit a data
   package, query the verified price, register and deregister
   feeds, update signer sets.
4. Provide an IDL for the adaptor program and the canonical
   oracle price account standard (re-exported from RFP-019, not
   forked), using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Return clear, actionable error messages for all failure modes:
   stale data package, signer-threshold not met, signer not in
   authorised set, asset identifier mismatch, malformed package,
   invalid signature, zero or negative price.
6. Provide a **reference consumer program**: a minimal LEZ program
   (or equivalently a documented program-side code snippet plus
   tests) that demonstrates the recommended consumer-side
   integration pattern for reading the canonical price account
   populated by this adaptor. The reference must show: reading
   price and timestamp from the account, rejecting prices older
   than the consumer's chosen `maxAge`, refusing to act on a
   price whose dispute flag is set (the dispute flag is owned by
   RFP-019's circuit breaker), and the recommended response when
   a price is unavailable (typically: refuse the action, do not
   fall back to an unsafe default). This is a guidance artefact
   for downstream consumer protocols (RFP-008, RFP-013, RFP-004),
   not a production product on its own.

#### Reliability

1. A price read is read-only and never modifies adaptor state.
2. Feed registration is atomic: partial failure leaves existing
   registrations intact.
3. Signature verification is deterministic: given the same data
   package and signer set, the verification result is the same.

#### Performance

1. End-to-end signature verification and price publication for a
   single 3-of-N RedStone data package must complete within a
   single LEZ transaction at the per-transaction compute and proof
   budget in force on LEZ at delivery time.
2. Cost measurement is a primary deliverable, not a side report.
   The applicant must measure and document, for the RISC-V
   in-program verification path: per-signer ECDSA recovery cost
   (compute units, RISC0 proof time, RISC0 proof size), keccak256
   hashing cost, package decoding cost, signer-set membership
   check, canonical price account write, and feed registration.
   Numbers must be reproducible from the test suite.
3. Document the cost delta between the in-program path and a
   hypothetical native ECDSA + keccak256 precompile, using existing
   per-chain reference points (for example, the RedStone EVM
   end-to-end gas range of 50K to 100K, and the per-recovery cost
   profile on chains that expose a native primitive). The delta
   informs whether a follow-on precompile RFP is warranted.

#### Supportability

1. The adaptor program is deployed and tested on LEZ
   devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI; CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test. The test
   suite must include: valid signature acceptance, invalid
   signature rejection, signer-threshold enforcement (M-of-N,
   including boundary cases), stale-package rejection (`maxAge`),
   asset-identifier mismatch rejection, zero or negative price
   rejection, and feed registration / signer-set update transitions.
4. A README documents end-to-end usage: deployment steps, program
   addresses, initial XMR/USD and ZEC/USD feed registrations, and
   step-by-step instructions for submitting data packages and
   querying prices via CLI and mini-app.
5. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for
   submitting RedStone data packages and reading verified prices,
   **plus a "Recommended Consumer Pattern" section** that walks a
   downstream protocol developer through the reference consumer
   program from Usability #6: staleness handling, dispute-flag
   handling, behaviour when no valid non-disputed price is
   available, and the recommended pairing with the on-chain TWAP
   tier from RFP-019 for divergence checking.
6. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
7. Provide Figma designs or equivalent for the mini-app GUI
   (off-chain feed dashboard).

#### + Adaptor Security

1. The adaptor must reject any data package whose recovered
   signer is not in the configured authorised signer set for the
   requested feed.
2. The signer set must be updatable only by the program owner;
   the update path itself must be tested.
3. The minimum recommended `maxAge` for production use is
   documented, with a manipulation analysis covering signer
   compromise, replay of stale packages, and signer-set update
   delays.

### Soft Requirements

1. Multi-feed batched verification: amortise calldata and
   signature recovery overhead across multiple feeds in a single
   instruction (analogous to Pyth's Perseus amortisation).
2. Circuit-breaker integration test against the on-chain TWAP
   tier from RFP-019 once the TWAP program is available: confirm
   that divergence between the RedStone-published price and the
   TWAP-published price triggers the dispute flag as specified in
   RFP-019.

### Out of Scope

The following are explicitly excluded from this RFP and addressed
elsewhere:

- The on-chain TWAP tier and the canonical oracle price account
  standard are owned by [RFP-019](./RFP-019-twap-oracle.md). This
  RFP populates the standard, it does not define it.
- A Pyth adaptor. Pyth depends on Wormhole on LEZ and is deferred
  to a future RFP. Higher publisher counts and confidence
  intervals (which RedStone does not natively expose) come with
  that adaptor.
- Adaptors for other off-chain oracles (Chainlink, DIA, Chronicle,
  Switchboard, Supra). None of these match the combination of
  privacy-asset coverage, single-primitive verification, and
  bridge independence that motivates this RFP. Future RFPs may
  add them.
- Pull-mode reads from inside private execution. A private
  transaction that wants to verify a secp256k1 signature inline
  cannot do so without forfeiting batching benefits or breaking
  privacy (see Design Rationale). Private composability is via
  reading the public price account that the push-mode aggregator
  writes to.
- Adding a secp256k1 ECDSA + keccak256 precompile to LEZ. The
  RISC-V in-program path is the deliverable here. A precompile
  becomes a candidate for a follow-on RFP if and only if the cost
  measurement in this RFP shows the in-program path is too
  expensive for production cadence.
- The choice between LSC/USD direct and LGS/USD + LGS/LSC
  composite for the LSC stablecoin
  ([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)). That
  is a business decision left to the RFP-013 implementer.

## ⚠ Platform Dependencies

### Hard blockers

None at the runtime level. The adaptor builds on the LEZ runtime
as it stands today (RISC-V zkVM on RISC0, public-execution mode,
public account storage). Signature verification runs as in-program
code; no new precompile or syscall is required to deliver the
adaptor.

### Cost-conditional follow-on (not a blocker for this RFP)

#### secp256k1 ECDSA + keccak256 precompile in public-execution mode

If the cost measurement deliverable shows that in-program ECDSA
recovery and keccak256 hashing in RISC0 are too expensive for the
push-mode aggregator's production cadence, a follow-on RFP can
propose adding a precompile (or accelerated host function) to LEZ
for use by public-execution programs. That RFP would substitute
for the in-program verification path in this adaptor via the
localised swap-out described in the Functionality requirements.
The precompile would be public-mode only; private execution paths
are unaffected because they do not call this primitive.

The LEZ runtime team has noted that supporting a secp256k1
primitive raises a broader set of design questions (nullifier
tracking for replay, privacy-circuit branching to support
Ethereum-signed private accounts, identifier-flow / wallet
implications) that are not blockers for the narrow oracle use of
the precompile but should be acknowledged. Those questions can be
scoped out of the follow-on or addressed in a separate runtime
RFP, depending on appetite.

### Soft blockers

#### RFP-019 (canonical oracle price account standard)

This RFP populates the canonical price account standard defined in
[RFP-019](./RFP-019-twap-oracle.md). If RFP-019 has not landed
when this RFP is delivered, the applicant defines a
forward-compatible minimal struct (see Design Rationale).

#### Event emission (LP-0012)

Analytics and monitoring benefit from structured on-chain events
for price updates, feed registrations, and signer-set changes.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently
**open**.

## 👤 Recommended Team Profile

Team experienced with:

- Oracle or DeFi protocol infrastructure development
- Cryptographic verification (secp256k1 ECDSA recovery, keccak256
  hashing, calldata parsing, signer-set management)
- LEZ / RISC0 program development; in particular, comfort writing
  and measuring elliptic-curve and hash-function code in RISC-V
  programs proved by RISC0 (cost characterisation experience is a
  strong signal, since cost measurement is a primary deliverable)
- RedStone's data-package format, EVM connector, or Solana
  connector (any prior integration is a strong signal)
- Smart-contract security auditing (signer compromise, replay
  attacks, signer-set update races)

## ⏱ Timeline Expectations

Estimated duration: **6 to 10 weeks**.

The adaptor has no hard runtime dependencies; it builds on LEZ as
it stands today. The canonical price account standard is a soft
dependency on RFP-019 with a documented fallback. The cost
measurement deliverable resolves the open question of whether
in-program ECDSA + keccak256 in RISC0 is fast enough for the
push-mode aggregator at production cadence; if not, a follow-on
RFP for a secp256k1 precompile becomes the optimisation path,
with this adaptor as the immediate consumer.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (consumer of price feeds; private swaps consume the
  RedStone-published prices)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (primary consumer of price feeds)
- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
  (LSC stablecoin; either Path A direct LSC/USD or Path B
  composite uses this adaptor)
- [RFP-019 — On-Chain TWAP Oracle](./RFP-019-twap-oracle.md)
  (defines the canonical oracle price account standard and
  circuit-breaker interface)
- [Appendix: Oracle Ecosystem](../appendix/oracle-ecosystem.md)
- [RedStone Documentation](https://docs.redstone.finance/)
- [RedStone token registry](https://github.com/redstone-finance/redstone-api/blob/main/docs/ALL_SUPPORTED_TOKENS.md)


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
