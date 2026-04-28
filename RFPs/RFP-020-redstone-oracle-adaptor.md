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

Build a RedStone off-chain oracle adaptor for LEZ: a SVM program
that verifies RedStone-signed data packages from instruction
calldata, exposes the resulting prices through the canonical oracle
price account standard defined in
[RFP-019](./RFP-019-twap-oracle.md), and supports day-one delivery
of XMR/USD and ZEC/USD feeds. RedStone's data packages are signed
with secp256k1 + keccak256 by its data nodes, recoverable on LEZ
via the SVM native secp256k1 precompile, with no cross-chain bridge
or Wormhole dependency. This RFP is scoped to the RedStone adaptor
only; on-chain TWAP is in RFP-019, and a Pyth adaptor (which
depends on Wormhole on LEZ) is deferred to a future RFP.

(Scope note: this RFP is about asset-price oracles for DeFi
applications. It is unrelated to the RLN service-attestation oracle
work on the anon-comms roadmap.)

## 🔥 Why This Matters

Logos's thesis is private DeFi: assets, applications, and users
that the broader web3 stack does not yet serve well. Privacy
collateral, in particular Monero (XMR) and Zcash (ZEC), is the
clearest day-one differentiator and the most direct path to
attracting privacy-aligned developers and capital to LEZ. Building
on top of those assets, the LSC stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), the
privacy-preserving DEX
([RFP-004](./RFP-004-privacy-preserving-dex.md)), wrapped privacy
assets, and other cross-chain primitives all require a USD
reference price for XMR and ZEC. Without one, none of those
applications can ship.

Across the surveyed off-chain oracle providers, RedStone is the
only one that combines: support for both XMR and ZEC in its public
token registry, an SVM-portable connector that recovers signatures
via the native secp256k1 precompile, no cross-chain bridge
requirement, and a self-serve deployment path that does not require
an oracle-team business engagement. Pyth covers both feeds and adds
higher publisher counts and confidence intervals, but is gated on
Wormhole integration on LEZ; it should land as a fast-follow in a
future RFP. Chainlink is permissioned and not self-serve. DIA
Lumina is permissionless but requires bespoke per-chain
deployment. See
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
  will have more robust CEX liquidity once the network reaches
  scale; external oracles for LGS/USD are battle-tested. Cons: the
  LGS/LSC TWAP becomes the manipulation bottleneck, which is
  exactly the low-liquidity vulnerability the TWAP RFP raises.

This RFP does not pick A or B. It provides the off-chain oracle
half of either path: USD prices for LSC, LGS, XMR, ZEC, and any
other asset RedStone supports. RFP-019 provides the TWAP half
needed for Path B. Whichever path the LSC implementer ultimately
chooses for production (informed by the realities of CEX liquidity
and adoption), this adaptor remains the swap-in for the off-chain
component.

## 🏗 Design Rationale

### Public oracle execution

The adaptor runs as a public LEZ execution with no confidential
state. Signature verification, data-package decoding, and price
publication are all visible to any caller. Any LEZ dapp can read
the same canonical price. Confidential execution is reserved for
application-layer protocols that consume oracle prices (for
example, private DEX swaps in
[RFP-004](./RFP-004-privacy-preserving-dex.md)); the price feed
itself stays public.

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
2. **SVM-portable verification with existing primitives.**
   RedStone's signature scheme is plain m-of-N secp256k1 ECDSA
   (typically 3-of-N), recoverable via the SVM secp256k1 SigVerify
   precompile at roughly 20K to 35K compute units per update. No
   new cryptographic precompile is required on LEZ to host the
   adaptor. Pyth's full 13-of-19 Wormhole VAA verification is
   roughly 87K compute units per signature step plus Merkle proof
   on top of that, even before Wormhole guardian-set tracking is in
   place.
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
struct using SVM's append-friendly account-data conventions, so
that a later RFP-019 release can extend the struct without
breaking consumers.

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

1. Implement an SVM program that accepts signed RedStone data
   packages from instruction calldata, recovers each signer's
   public key using the SVM secp256k1 SigVerify precompile, and
   verifies that the recovered public keys match the configured
   set of authorised RedStone data nodes for the requested feed.
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

#### Reliability

1. A price read is read-only and never modifies adaptor state.
2. Feed registration is atomic: partial failure leaves existing
   registrations intact.
3. Signature verification is deterministic: given the same data
   package and signer set, the verification result is the same.

#### Performance

1. End-to-end signature verification and price publication for a
   single 3-of-N RedStone data package must complete within a
   single LEZ transaction.
2. Document the compute unit (CU) cost of: signature verification
   per signer, package decoding, signer-set membership check,
   canonical price account write, and feed registration. LEZ's
   per-transaction compute budget may change during testnet.

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
   submitting RedStone data packages and reading verified prices.
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
  privacy-asset coverage, SVM-portable verification, and bridge
  independence that motivates this RFP. Future RFPs may add them.
- The choice between LSC/USD direct and LGS/USD + LGS/LSC
  composite for the LSC stablecoin
  ([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)). That
  is a business decision left to the RFP-013 implementer.

## ⚠ Platform Dependencies

### Hard blockers

#### SVM secp256k1 precompile

The adaptor relies on the SVM secp256k1 SigVerify precompile to
recover signers from RedStone's secp256k1 + keccak256 signatures.
LEZ inherits this precompile from its Solana base; no new
precompile is required.

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
- Cryptographic verification (secp256k1 ECDSA recovery, calldata
  parsing, signer-set management)
- Solana or SVM program development (Anchor or native), including
  use of the secp256k1 SigVerify precompile
- RedStone's data-package format, EVM connector, or Solana
  connector (any prior integration is a strong signal)
- Smart-contract security auditing (signer compromise, replay
  attacks, signer-set update races)

## ⏱ Timeline Expectations

Estimated duration: **6 to 10 weeks**.

The adaptor has no hard external dependencies beyond the SVM
secp256k1 precompile (already present on LEZ); the canonical price
account standard is a soft dependency on RFP-019 with a documented
fallback.

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
