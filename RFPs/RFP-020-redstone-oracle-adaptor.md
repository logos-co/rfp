---
id: RFP-020
title: RedStone Off-Chain Oracle Adaptor for LEZ
tier: M
status: closed
dependencies:
  - id: RFP-001
    reason: Provides the standardised admin authority library that registers feeds and updates authorised signer sets, as specified in the Functionality requirements.
category: Developer Tooling & Infrastructure
---

# RFP-020 — RedStone Off-Chain Oracle Adaptor for LEZ

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

Build a RedStone off-chain oracle adaptor for LEZ: a public-mode LEZ program
that verifies RedStone-signed data packages, exposes the resulting prices
through the canonical oracle price account standard defined in
[RFP-019](./RFP-019-twap-oracle.md), and supports day-one delivery of BTC/USD,
ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds. RedStone's data packages are
signed with secp256k1 + keccak256 by its data nodes; verification on LEZ runs as
in-program code inside the RISC-V zkVM (no cross-chain bridge, no Wormhole
dependency). The adaptor delivers **both modes** that the RedStone connector
model supports on a public runtime: a **push-mode aggregator** (a public-mode
program verifies signatures on the write side, stores the result in a public
price account, and consumers read the slot) and **public-mode pull** (a public
consumer transaction carries a signed RedStone payload and verifies it inline
against the same authorised signer set, without writing a shared price account).
Both modes share a single verification path so the cost profile and audit
surface are common.

LEZ is RISC0-based, so any signature scheme can be implemented in program code.
Early prototype work on in-program secp256k1 ECDSA verification inside RISC0
([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench))
shows the verification is slow enough that **pull-mode reads from inside a
private transaction are not feasible on consumer hardware** (a private consumer
would spend several minutes generating the proof for each read), absent a
RISC0-specific signature-verification accelerator. Pull mode is therefore
delivered for **public execution only**: a public consumer transaction can
verify a signed payload inline at the cost measured in this RFP, while
private-execution composition continues to go via the public price account
written by the push aggregator. Cost measurement remains a primary deliverable
for both modes: the push-mode write side, where amortisation across all
downstream reads can make in-program verification workable, and the public-mode
pull path, where the per-read cost is paid by every consumer transaction. If the
measured public-mode cost is unacceptable on either path, the measurement
becomes the input to a follow-on RFP that proposes adding a secp256k1 ECDSA +
keccak256 precompile to LEZ.

### Scope

In scope:

- The RedStone off-chain oracle adaptor on LEZ, delivered in two modes:
  - **Push-mode aggregator** (public-mode program owns a canonical public price
    account).
  - **Public-mode pull** (a consumer-side verification library / on-chain helper
    that verifies a signed payload inline inside a public consumer transaction).
- Day-one BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds, registered and
  exercised on LEZ devnet/testnet in both modes.
- Cost measurement of the in-program RISC-V verification path as a primary
  deliverable, covering both the push-aggregator write side and the public-mode
  pull per-read cost.
- Operating the push-mode feed infrastructure for the day-one feeds under an
  SLA, from software readiness through March 2028 (see Servicing).

Out of scope at the Overview level (full list under Out of Scope below):

- The on-chain TWAP tier and the canonical oracle price account standard, owned
  by [RFP-019](./RFP-019-twap-oracle.md).
- Pull-mode reads from inside *private* execution (public-mode pull is in scope
  above): blocked on either a RISC0 zkVM circuit-level accelerator (e.g. a
  future `risc0-ecdsa` extension that lowers in-circuit verification cost) or a
  different upstream signature scheme that admits acceptable in-circuit cost on
  RISC0; neither exists today.

## 🔥 Why This Matters

Private DeFi is what LEZ is positioned to support, and reliable USD reference
prices for privacy collateral, in particular Monero (XMR) and Zcash (ZEC), are a
necessary step. The reflexive stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), the privacy-preserving
DEX ([RFP-004](./RFP-004-privacy-preserving-dex.md)), wrapped privacy assets,
and other cross-chain primitives all need those reference prices to function.

The on-chain TWAP tier in [RFP-019](./RFP-019-twap-oracle.md) is not sufficient
on its own for the day-one asset list. TWAP security scales linearly with pool
depth: on a new chain where liquidity is thin, a validator controlling two
consecutive blocks can manipulate the accumulator at a cost roughly equal to the
round-trip swap fees and price impact, which on a $1M pool is cheap (see
[Appendix: TWAP Manipulation Vectors](../appendix/oracle-ecosystem.md)). This
applies to every pair on a new chain, not just to privacy assets. More
structurally, TWAP only produces a price for pairs that exist as pools on LEZ.
An off-chain feed is the only way to get an XMR/USD or ZEC/USD reference on
chain at all, and pairing it with TWAP for the pairs where TWAP does work is the
production norm for layered oracle defence.

Across the surveyed off-chain oracle providers, RedStone is the only one that
combines: support for both XMR and ZEC in its public token registry, a portable
connector pattern (single secp256k1 ECDSA + keccak256 verification path that
works the same on every host chain), no cross-chain bridge requirement, and a
self-serve deployment path that does not require an oracle-team business
engagement. Pyth covers both feeds and adds higher publisher counts and
confidence intervals, but is gated on Wormhole integration on LEZ; it should
land as a fast-follow in a future RFP. Chainlink is permissioned and not
self-serve. DIA Lumina is permissionless but requires bespoke per-chain
deployment. See
[Appendix: Oracle Ecosystem, Privacy-Asset Feed Availability](../appendix/oracle-ecosystem.md)
for the full coverage matrix.

The combination of "private DeFi needs XMR and ZEC" and "RedStone is the only
path that is self-serve on LEZ today" makes this the priority off-chain oracle
integration for LEZ.

### A building block in a layered oracle stack

Neither an off-chain feed nor an on-chain TWAP is a complete oracle on its own;
both have known failure modes and the production norm in DeFi is to layer them.
This RFP delivers the off-chain adaptor as a swappable building block in that
layered stack: production-grade code on its own terms, paired with the TWAP tier
from [RFP-019](./RFP-019-twap-oracle.md) on the consumer side. Consuming
protocols (the reflexive stablecoin in
[RFP-013](./RFP-013-reflexive-stablecoin-protocol.md), the lending market in
[RFP-008](./RFP-008-lending-borrowing-protocol.md), the DEX in
[RFP-004](./RFP-004-privacy-preserving-dex.md)) compose these pieces according
to their own production-security choices, with the canonical price account
standard from RFP-019 keeping swap-out cheap if those choices change later.

## 🏗 Design Rationale

### Push aggregator and public-mode pull

The adaptor runs entirely in public execution and ships two consumption patterns
over a single verification path:

1. **Push-mode aggregator.** A public-mode LEZ program owns a canonical price
   account. A relayer (or any caller) submits a signed RedStone payload; the
   program verifies signatures and the M-of-N threshold and writes price plus
   timestamp to the public price account. Cost is paid once per update and
   amortises across all downstream reads, public and private. Private-execution
   consumers compose with the price by reading the public account.
2. **Public-mode pull.** A public consumer transaction carries a signed RedStone
   payload as part of its own input and runs the same verification path inline
   (as a library call from the consumer program), without touching the
   aggregator account. The verified price is consumed in the same transaction
   and is **not** written through to the canonical price account. Cost is paid
   per consumer transaction; the price is fresh at the moment of consumption.

Both modes share the same in-program secp256k1 ECDSA + keccak256 verification,
the same authorised signer-set configuration, and the same data-package decoder.
The aggregator is a thin wrapper that calls the shared verifier and writes a
canonical price account; the public-mode pull helper exposes the same verifier
as a library that a consumer program can call. This keeps the cost profile and
audit surface common between the two.

Pull from inside *private* execution is a different story, and is foreclosed on
LEZ today regardless of which mode the caller picks. LEZ is a RISC-V zkVM built
on RISC0; any code that runs inside a private transaction has to be expressible
inside the RISC-V zkVM circuit, so a private transaction that wants to verify a
secp256k1 ECDSA signature has two options, both unappealing: verify the
signature inside the privacy circuit (forfeits the batching benefits that make
ZK proof amortisation work; the prototype work referenced in the Overview shows
several minutes of proof time per read on consumer hardware), or place the
signature in the transaction journal where it is publicly disclosed (breaks the
privacy of the transaction). Neither option preserves both efficiency and
privacy. Private-execution programs therefore compose with the feed by reading
the public price account written by the push aggregator, not by carrying signed
payloads inline.

Public-mode pull is delivered because the LEZ consumer set includes
public-execution programs (administrative actions, public-mode market
operations, integration tests, and any consumer protocol that has chosen to run
a given action in public execution for its own reasons) for which the per-read
freshness of pull and the absence of a shared write-side dependency are real
value, not noise. Aggregator and pull are not redundant: pull removes the
shared-account update cadence as a dependency for any consumer that prefers to
pay per read; the aggregator removes the per-read verification cost for any
consumer (especially private-execution consumers) that prefers to amortise.
Shipping both lets each consumer protocol choose the trade-off, and the cost
numbers from the measurement deliverable give that choice a quantitative basis.

See
[Appendix: Oracle Ecosystem, Implications for LEZ](../appendix/oracle-ecosystem.md#implications-for-lez)
for the full four-shape analysis (trusted re-signer, FROST-BIP340 federation,
DLC-oracle extension, and the cost-conditional precompile path) that this
section condenses.

A LEZ-specific freshness pattern follows from the public / private execution
split. A user who needs a price fresher than the heartbeat's last update can
submit a public transaction that pushes a fresh signed payload to the aggregator
account, then submit a private transaction immediately after that reads the
just-updated public price. Verification cost is paid in the public path
(cheaper, especially under the precompile follow-on); the private transaction
does no signature work. This recovers pull mode's "fresh at transaction time"
property for private consumers without paying the in-circuit cost in the privacy
proof. The adaptor program already accommodates this: any caller can submit a
valid signed payload and the program writes if signatures and timestamps check
out.

The two-transaction split (public push, then private read) is distinct from the
rejected "put the signature in the journal" option. The signature is carried
only in the public push; the private transaction reads the resulting public
price account by address with no upstream signature in its calldata or journal,
so the private transaction's contents (assets, counterparty, amount) stay
private. This fits the adaptor's existing write path: the program already
accepts signed payloads from any caller and writes a public price account,
regardless of whether the caller is a relayer or an end user. There is a
residual linkability risk that the consumer needs to handle on its side, not the
adaptor's: an observer can correlate a public push from wallet X at time T with
a private transaction at time T plus epsilon and infer that the same actor is
consuming the just-pushed price. Mitigations (separate funding wallet for the
push, timing decorrelation, reliance on a heartbeat to mask single-purpose
pushes) are consumer-side production-security choices. The privacy story is
strictly better than journal-disclosed signatures (the private transaction's
body stays private) but not equivalent to a heartbeat-only push model.

### RISC-V verification path and the precompile question

This RFP implements signature verification in RISC-V program code, running
inside RISC0. There is no host primitive to call: the recovery is an in-program
ECDSA + keccak256 path written against existing Rust crates (k256 / sha3 /
equivalents) and proved by RISC0 along with the rest of the program.

This is the central technical bet of the RFP. Early prototype work on in-program
secp256k1 ECDSA verification inside RISC0
([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench))
is already enough to flag that the naive in-circuit path is slow on consumer
hardware: a private consumer attempting pull-mode verification would spend
several minutes generating the proof for each read. That rules out
private-execution pull mode in any practical sense for the in-program path,
absent a RISC0-specific signature-verification accelerator (e.g. a future
`risc0-ecdsa` extension or a secp256k1 precompile wired into the zkVM proving
system itself). The first concrete deliverable of this RFP refines this picture
for the two delivered modes: implement the verifier in RISC-V, run it on LEZ in
both the push aggregator and a representative public-mode pull consumer
transaction, and document the cost (compute units, proof time, proof size,
per-update / per-read bytes) for the per-signature recovery, the full 3-of-N
aggregator write, and the full 3-of-N pull-mode read. Characterise where each
lands relative to the production-cadence budget. The two modes amortise cost
differently and the measurement must keep them distinct: aggregator cost is paid
once per update and divided across all downstream reads; pull cost is paid by
every consumer transaction.

Three outcomes are possible from that measurement:

1. **Measured cost is acceptable for both modes.** The adaptor ships on the
   runtime as it stands. No runtime change required.
2. **Measured cost is acceptable for the push-mode aggregator but not for
   public-mode pull.** The adaptor still ships with both modes available (the
   pull library is documented as a fit only for low-frequency or high-value
   consumer transactions that can absorb the per-read cost), but the cost
   measurement becomes the input to the follow-on RFP in outcome 3.
3. **Measured cost is unacceptable in either mode.** The measurement becomes the
   input to a follow-on RFP that proposes adding a secp256k1 ECDSA + keccak256
   precompile to LEZ for use by public-execution programs. A precompile lives
   outside the ZK proof boundary and is invoked as native validator code, so the
   cost goes from "ZK-proven elliptic-curve operations" to "native ECDSA
   recovery + keccak", which is the cost profile RedStone's existing connectors
   assume on every other chain. The precompile is an optimisation path
   conditional on the measurement, not a precondition for this RFP; because the
   two modes share the verification core, the precompile lowers cost in both
   simultaneously.

The applicant should therefore design the verification path so that swapping in
a precompile in a later release is a localised change (a single trait
implementation or syscall wrapper), not a restructuring of the program.

The precompile path addresses public-mode cost (in both push and pull, via the
shared verification core); private-execution pull mode is foreclosed under it
for the same structural reason it is foreclosed under the in-program path (a
precompile lives outside the ZK proof boundary, so it is not callable from
inside a private transaction). If a signature scheme exists, or can be selected,
that yields acceptable in-circuit cost on RISC0 (a RISC0-friendly hash and curve
combination, or a different signature primitive that admits cheaper in-circuit
verification), pull-mode reads from inside private execution become reachable as
a third deployment mode alongside the push aggregator and public-mode pull this
RFP delivers. Identifying or building such a scheme, and either modifying an
existing oracle network to publish in it or standing up a new publisher set that
does, is the subject of a possible separate follow-on RFP. It is independent of
the public-mode precompile question and out of scope for this RFP. Whether such
a follow-on is worth pursuing depends on whether any consumer protocol actually
needs private-execution pull, which is not yet confirmed: consumer protocols may
already have design reasons to keep specific actions in public transactions,
with the reflexive stablecoin in
[RFP-013](./RFP-013-reflexive-stablecoin-protocol.md) as one concrete example.

### Why RedStone first

Three reasons specific to LEZ's constraints:

1. **Privacy-asset coverage with no bridge.** Both XMR and ZEC are in RedStone's
   public token registry, and the RedStone connector pattern is fully
   self-serve: deployment does not require an oracle-team engagement, a bridge,
   or a per-chain registration step. As far as we can see from the public SDK
   ([`@redstone-finance/sdk` on npm](https://www.npmjs.com/package/@redstone-finance/sdk))
   and the
   [RedStone Pull docs](https://docs.redstone.finance/docs/dapps/redstone-pull/),
   consuming the live data-package gateways requires no account, API key, or
   signup at the technical level; use is governed by the public
   [RedStone Terms of Use](https://redstone.finance/terms-of-use) (acceptance
   implied by use). Whoever uses or runs the resulting software is the party
   bound by those Terms; see "Operator-side T&C considerations" below. LEZ can
   deploy and exercise these feeds without touching any external infrastructure.
   Pyth and DIA both cover the same assets but require either Wormhole (Pyth) or
   bespoke per-chain deployment (DIA Lumina) before they work on a new chain.
2. **Single verification primitive, no bridge.** RedStone's signature scheme is
   plain m-of-N secp256k1 ECDSA over keccak256 (typically 3-of-N). Verification
   on LEZ is in-program ECDSA recovery and keccak256 hashing inside RISC0; the
   cost profile is the open variable this RFP measures (see "RISC-V verification
   path and the precompile question"). Pyth's full 13-of-19 Wormhole VAA
   verification is heavier in two ways: it adds a Merkle proof on top of more
   signatures, and it presupposes a Wormhole guardian-set tracking program on
   LEZ that does not yet exist. RedStone has neither cost.
3. **Independent of LEZ's external integration timeline.** Choosing RedStone
   first decouples the oracle layer from when Wormhole on LEZ is decided. Pyth
   then fast-follows in a future RFP, contributing higher publisher counts
   (especially the roughly 80+ on XMR/USD versus RedStone's smaller per-feed
   roster) and confidence intervals that RedStone does not natively expose.

See
[Appendix: Oracle Ecosystem, Signature Verification Schemes](../appendix/oracle-ecosystem.md)
for the full per-scheme analysis and citations.

### Conformance to the canonical price account standard

The canonical oracle price account standard is owned by RFP-019 (see "LEZ oracle
data standard" in that RFP's Design Rationale). The RedStone adaptor populates
the same struct as the on-chain TWAP source: `base_asset`, `quote_asset`, price,
timestamp, source identifier, and confidence interval (zero, since RedStone does
not publish one). Consuming protocols query a single data layout and remain
agnostic to whether the price came from TWAP, RedStone, or any future provider;
cross-source policy lives in the consumer per RFP-019, Design Rationale
("Multi-source coexistence").

If RFP-019 has not yet shipped the canonical struct when this RFP is delivered,
the team must define a forward-compatible minimal struct using append-friendly
account-data conventions, so that a later RFP-019 release can extend the struct
without breaking consumers.

### HTTPS data path: centralisation and censorship

Signed RedStone data packages are fetched from the RedStone Data Distribution
Layer (DDL) over **HTTPS, not P2P**. The default gateway URLs are DNS-resolved
endpoints under `*.redstone.finance` and `*.redstone.vip` (see the SDK source at
`packages/sdk/src/data-services-urls.ts` and the appendix section
"Infrastructure Requirements for External Oracles on LEZ"). This applies whether
the data reaches LEZ via the relayer (push) or inline as transaction calldata
(pull). Two consequences follow:

- **Censorship surface.** A gateway operator (and its upstream network) can
  withhold responses to specific clients, geographies, or asset IDs. On-chain
  signature verification protects against falsified data, not against withheld
  data: a censored consumer cannot obtain a signed payload, and any transaction
  depending on it reverts.
- **Liveness dependency.** DNS, TLS validity, and HTTPS availability at
  RedStone's hosts (AWS, GCP) all become LEZ oracle liveness dependencies.
  Regional outages at these providers can take the feed offline.
- **Privacy leak in pull mode.** Pull-mode consumers fetch the signed data
  package directly from the RedStone DDL gateway before submitting it inline
  with their on-chain transaction. This reveals the consumer's IP (and timing,
  and which asset pair was requested) to the gateway operator *before* the
  on-chain action. Public-mode pull is in scope in this RFP, so this exposure is
  a real caveat for any consumer that picks it; the push aggregator confines the
  same exposure to the relayer operator (the relayer's IP is visible to
  RedStone, but downstream aggregator consumers read from the public price
  account on LEZ with no gateway contact). The pull-mode SDK and reference
  consumer must document this trade-off, and consumer protocols choosing pull
  should consider IP-anonymising fetch paths (Tor, mixnets, gateway proxies) or
  accept the exposure as appropriate to their threat model.

Implementers must document, as part of the relayer operator journey, the
mitigations the operator chooses: multiple parallel gateway queries (SDK
default), the use of RedStone private gateways via the
`OVERRIDE_DIRECT_CACHE_SERVICE_URLS` mechanism where available, and recent-price
fallback policy if no gateway responds within the operator's tolerance. None of
these mitigations eliminate the structural centralisation; they reduce specific
failure modes.

This caveat is inherent to RedStone's design and to every oracle in the survey
that uses an HTTPS portal (Pyth Hermes has the same property). It is not an
objection to RedStone; it is the trade-off LEZ accepts by adopting any
HTTPS-served signed-data oracle, and must be communicated to consumer protocols
so they can size their liveness assumptions accordingly.

### Operator-side T&C considerations

This RFP scopes both the delivery of *software* (the adaptor program, the
relayer module, the SDK and consumer-pattern documentation) and a defined period
of *operating* that software (see Servicing). During the operating period the
awarded team runs the relayer that fetches data packages from
`*.redstone.finance` and `*.redstone.vip`, so for that period the awarded team
is itself a party bound by the
[RedStone Terms of Use](https://redstone.finance/terms-of-use) and must abide by
them. Once operation later moves to another operator, whoever runs the software
becomes the bound party.

The implementer's documentation **must** surface the Terms of Use to relayer
operators as part of the operator journey (see Functionality #7 and
Supportability), so any operator deploying the relayer, including the awarded
team during the operating period and any independent operator that later runs
the feeds, is on notice that they are the party bound by the Terms. Whether the
operating engagement requires a commercial agreement with RedStone is for the
awarded team to determine and comply with as the operating party.

### Fee structure

Verification cost lands differently in the two modes. In push mode, the on-chain
verification cost is paid once per update by whoever submits the signed data
package to the aggregator, and is amortised across all downstream reads (public
and private); RedStone itself does not publish prices on-chain, so "whoever pays
for an update" in practice means whoever runs (or pays for) the relayer. In
public-mode pull, the verification cost is paid by each consumer transaction on
every read, with no shared amortisation but no relayer dependency either. The
adaptor design does not need to fund a dedicated node operator pool for either
mode; during the operating period the awarded team runs the push-mode relayer
under the Servicing requirements, and the intended end state past that period is
that independent ecosystem operators run it by choice, typically as a
loss-leader alongside their other paid services.

Beyond that structural point, this RFP does not prescribe a fee model.
Downstream users of the adaptor (consuming protocols, relayer operators,
deployers) are free to handle fees in whatever way fits their product:
subsidised by the consuming protocol, charged per read, charged per update,
routed to a treasury, burned, or left at zero. The adaptor program itself should
not bake in policy that forecloses these choices.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a public-mode LEZ program (push-mode aggregator) that accepts
   signed RedStone data packages. Structure the verification path so that
   swapping the in-program signature validation for a future host primitive
   (precompile or syscall) is a localised change.
2. The verification routine must be packaged as a library callable both from
   this program (push mode) and from third-party consumer programs (public-mode
   pull, Functionality #8), so a single audited implementation backs both modes.
3. Verify the M-of-N signer threshold for each feed (configurable at
   registration; default 3-of-N consistent with RedStone's reference parameters)
   and reject any data package that does not meet the threshold.
4. Decode the RedStone data package format (asset identifier, value, timestamp,
   signer set) and reject any package whose timestamp is older than a
   configurable `maxAge`, whose value is zero, negative, or otherwise invalid,
   or whose asset identifier does not match the registered feed.
5. Publish the verified price into a canonical oracle price account conforming
   to the standard defined in RFP-019. The adaptor must populate `base_asset`,
   `quote_asset`, price, timestamp, source identifier (a constant identifying
   RedStone), and confidence interval (zero; RedStone does not publish
   confidence intervals in its standard data packages).
6. An admin authority (per RFP-001, integrated via the SPEL framework) can
   register new RedStone feeds (by asset identifier, M-of-N threshold, and
   authorised signer set), update an existing feed's signer set on RedStone
   roster changes, and deregister feeds.
7. BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds must be registered on
   LEZ devnet/testnet as part of the deliverable.
8. **Relayer module.** Provide a relayer service that fetches signed RedStone
   data packages from the RedStone Data Distribution Layer gateways (see
   Appendix, "Infrastructure Requirements for External Oracles on LEZ") and
   submits them to the LEZ adaptor. The relayer must be implemented as a **Logos
   module accompanied by a Logos Core headless CLI/daemon**, so that operators
   (RedStone, the Logos ecosystem, or consuming protocols) can run it as a
   standalone long-running process without requiring a user-facing app. The
   daemon must support: configurable `dataServiceId` (default
   `redstone-primary-prod`), configurable feed list and update triggers per feed
   (heartbeat interval and deviation threshold), parallel querying of multiple
   DDL gateways with first-success selection, retry and back-off on transient
   gateway or LEZ errors, structured logging of submitted updates and rejections
   (with the on-chain rejection reason where available), wallet-balance
   monitoring, and a clean shutdown path. Document the operator journey
   end-to-end: install, configure, run, monitor.
9. **Public-mode pull verification library.** Expose the verification path from
   Functionality #1 as a reusable library that a public-mode consumer program
   can call inline. Given a signed RedStone payload, a configured
   `(dataServiceId, feedId, authorised signer set, M-of-N threshold, maxAge)`,
   and the consumer's expected `(base_asset, quote_asset)`, the library returns
   the verified price and timestamp on success or a typed error on failure (same
   failure modes as the aggregator: invalid signature, signer not in set,
   threshold not met, stale package, asset mismatch, malformed package, zero or
   negative value). The library must not depend on the aggregator's price
   account; a consumer using only pull must be able to integrate without
   registering a feed against the aggregator program. Document the consumer-side
   fetch path (querying the RedStone DDL gateways for a signed payload before
   sending the consumer transaction) and the gateway-IP privacy caveat from
   Design Rationale ("HTTPS data path").

#### Usability

1. Provide an SDK that can be used to build Logos modules for: submitting
   RedStone data packages to the push aggregator; reading verified prices from
   the canonical price account; and consuming the public-mode pull verification
   library (Functionality #8) from a public consumer program. The SDK must
   expose both modes through ergonomic helpers so a developer can switch between
   them without changing the payload-handling code.
2. Provide a Logos mini-app GUI (off-chain feed dashboard) with local build
   instructions, downloadable assets, and loadable in Logos app (Basecamp). The
   dashboard must display, for the push aggregator: live prices for each
   registered feed, the configured signer set, the current M-of-N threshold, the
   latest data-package timestamp, and the staleness of each feed. It must
   additionally provide a public-mode pull dry-run panel that fetches a signed
   payload from the DDL for a chosen `(dataServiceId, feedId)` and shows what
   the pull verification library would return on-chain (verified price and
   timestamp, or typed error), so developers and operators can exercise the pull
   path without submitting a consumer transaction.
3. Provide a CLI that covers core functionality for both modes: submit a data
   package to the push aggregator, query the verified price from the canonical
   price account, register and deregister feeds, update signer sets, and (for
   pull mode) fetch a signed payload from the RedStone DDL and run an off-chain
   dry-run of the public-mode pull verification path for a given
   `(dataServiceId, feedId, base_asset, quote_asset, maxAge)` tuple. The dry-run
   output must report the same typed error codes the on-chain pull library
   returns.
4. Provide an IDL for the adaptor program and the canonical oracle price account
   standard (re-exported from RFP-019, not forked), using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Enhance the SPEL framework to easily hook in pull and push feed mode in new
   programs.
6. Return clear, actionable error messages for all failure modes: stale data
   package, signer-threshold not met, signer not in authorised set, asset
   identifier mismatch (`base_asset` or `quote_asset` does not match the
   registered feed), malformed package, invalid signature, zero or negative
   price.
7. Provide **two reference consumer programs**, one per mode (or one reference
   program with two clearly separated entry points), each a minimal LEZ program
   (or equivalently a documented program-side code snippet plus tests) that
   demonstrates the recommended consumer-side integration pattern:
   - **Aggregator-read reference.** Reads the canonical price account populated
     by the push aggregator. Must show: verifying the
     `(base_asset, quote_asset)` pair matches the consumer's expectation,
     reading price and timestamp from the account, rejecting prices older than
     the consumer's chosen `maxAge`, and the recommended response when a price
     is unavailable (typically: refuse the action, do not fall back to an unsafe
     default).
   - **Public-mode pull reference.** Calls the public-mode pull verification
     library from Functionality #8 with a signed payload carried in the consumer
     transaction's input. Must show: passing the expected
     `(base_asset, quote_asset)`, signer set, M-of-N threshold, and `maxAge` to
     the library, handling each typed error code, and the same "refuse on
     unavailable" pattern. Those must use SPEL as enhanced by item U5.

#### Reliability

1. A price read is read-only and never modifies adaptor state.
2. Feed registration is atomic: partial failure leaves existing registrations
   intact.
3. Upstream error with one feed does not impact the push mode for other feeds,
   when handled by the same node.
4. Temporary upstream errors leave the daemon in a recoverable manner, able to
   push again once upstream feeds are available.

#### Performance

1. End-to-end signature verification and price publication for a single 3-of-N
   RedStone data package must complete within a single LEZ public transaction at
   the per-transaction compute and proof budget in force on LEZ at delivery
   time. This applies to both modes: the push-aggregator write and a
   representative public-mode pull consumer transaction.
2. Cost measurement is a primary deliverable, not a side report. The applicant
   must measure and document, for the RISC-V in-program verification path, for
   **both modes** (push aggregator write and public-mode pull per-read):
   per-signer ECDSA recovery cost (compute units, RISC0 proof time, RISC0 proof
   size), keccak256 hashing cost, package decoding cost, signer-set membership
   check, and (push mode only) canonical price account write and feed
   registration. The measurement must explicitly report the per-read cost a
   public-mode pull consumer incurs, so a consumer protocol can decide which
   mode fits its cadence and budget. Numbers must be reproducible from the test
   suite.
3. Document the cost delta between the in-program path and a hypothetical native
   ECDSA + keccak256 precompile, using existing per-chain reference points (for
   example, the RedStone EVM end-to-end gas range of 50K to 100K, and the
   per-recovery cost profile on chains that expose a native primitive). Report
   the delta separately for the two modes (because the per-read pull path and
   the per-update aggregator path amortise the same precompile speed-up
   differently across the consumer set). The delta informs whether a follow-on
   precompile RFP is warranted.

#### Supportability

1. The adaptor program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI; CI must be green on the default branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test. The test suite must include,
   for **both modes** (push aggregator and public-mode pull where applicable):
   valid signature acceptance, invalid signature rejection, signer-threshold
   enforcement (M-of-N, including boundary cases), stale-package rejection
   (`maxAge`), asset-identifier mismatch rejection, zero or negative price
   rejection, and (push mode only) feed registration / signer-set update
   transitions. The pull-mode tests must exercise the library inside a
   representative public consumer program (the pull reference consumer from
   Usability #6), not just as a unit test of the verification function.
4. A README documents end-to-end usage for both modes: deployment steps, program
   addresses, initial BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feed
   registrations, step-by-step instructions for submitting data packages to the
   push aggregator and querying prices via CLI and mini-app, and step-by-step
   instructions for integrating the public-mode pull verification library into a
   consumer program (including how to fetch a signed payload from the DDL and
   how to handle each failure mode).
5. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for both modes:
   submitting RedStone data packages to the push aggregator, reading verified
   prices from the canonical price account, and integrating the public-mode pull
   verification library, **plus a "Recommended Consumer Pattern" section** that
   walks a downstream protocol developer through both reference consumer
   programs from Usability #6: `(base_asset, quote_asset)` verification,
   staleness handling, typed error handling for the pull path, behaviour when a
   price is unavailable, the gateway-IP privacy caveat for pull-mode consumers,
   and an example multi-source policy pairing either or both RedStone modes with
   the on-chain TWAP tier from RFP-019. The doc must include a "When to use push
   vs pull" decision guide grounded in the cost measurement deliverable, and
   must state explicitly that cross-source policy is owned by the consumer
   protocol, not by this adaptor.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
7. Provide Figma designs or equivalent for the mini-app GUI (off-chain feed
   dashboard).

#### Servicing

Beyond delivering the software, the awarded team must operate the live feed
infrastructure for a defined period. This makes the day-one feeds usable in
production without the ecosystem first having to stand up its own relayer
operations. Neither Chainlink nor RedStone publishes a numeric uptime SLA for
its public feeds, and RedStone's Terms of Use explicitly disclaim availability
("as-is / as-available"), so the targets below are commitments on the awarded
team's own operated relayer and monitoring stack, not figures inherited from
RedStone. The recommended numbers are grounded in incumbent-oracle parameters
and the closest published reliability figure we could locate; see
[Appendix: Oracle Service Levels and Feed Update Parameters](../appendix/oracle-ecosystem.md#oracle-service-levels-and-feed-update-parameters).

1. **Operating period.** Run the push-mode relayer/aggregator infrastructure for
   the BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds on the target
   network (devnet/testnet, then mainnet once available) from **software
   readiness through March 2028** (one year past the planned mainnet launch).
   Running the nodes means: operating the relayer daemon (Functionality #8),
   keeping wallets funded for update submission, and keeping the aggregator's
   authorised signer sets current with RedStone roster changes. The awarded team
   must not introduce any dependency on private configuration or bespoke
   infrastructure that would prevent another operator from reproducing the setup
   from the documented operator journey (Functionality #8), so that operation
   can move to another operator without a new deliverable.
2. **Feed uptime and liveness.** Maintain a documented per-feed availability
   target (recommended **99.9% monthly**, roughly a 43-minute monthly downtime
   budget, measured as the fraction of the heartbeat schedule for which a fresh,
   on-chain price within `maxAge` is available) and a bounded maximum staleness.
   99.9% is the closest published reliability figure the survey could locate;
   targets above it would rest on no published oracle precedent. The applicant
   proposes the exact numeric targets in the proposal; they become contractual
   on award.
3. **Update cadence.** Meet the configured per-feed heartbeat interval and
   deviation-threshold triggers, so a feed updates at least once per heartbeat
   and whenever the deviation threshold is crossed. Recommended defaults,
   grounded in incumbent-oracle parameters (Chainlink ETH/USD): a **0.5%
   deviation threshold and 1-hour heartbeat** for the volatile crypto/USD pairs
   (BTC, ETH, SOL, XMR, ZEC against USD), widening the band only where a
   specific asset proves too thin or volatile to sustain that cadence
   economically. Cadence targets are stated per feed and reported against
   (Servicing #5).
4. **Incident response.** Provide a documented incident-response process with
   target response and resolution times for feed outages (recommended:
   acknowledge within 1 hour, mitigate within 4 hours), an escalation path, and
   a named on-call contact. Post-incident, provide a root-cause summary for any
   outage that breaches the uptime target.
5. **Reporting.** Deliver periodic (at least monthly) operating reports covering
   per-feed uptime against target, cadence adherence, update and rejection
   counts, incidents and their resolution, and wallet-funding status. Provide
   the Logos ecosystem team with read access to the live monitoring dashboard
   (the mini-app off-chain feed dashboard from Usability #2, or an equivalent
   operator dashboard) for the operating period.

#### + Adaptor Security

1. The adaptor must reject any data package whose signer is not in the
   authorised signer set for the requested feed. This applies in both modes: the
   aggregator program for push, and the pull verification library (Functionality
   #9) for public-mode pull.
2. In push mode, the signer set stored on the aggregator program must be
   updatable only by the admin authority (per RFP-001, integrated via the SPEL
   framework); the update path itself must be tested. In pull mode, the signer
   set is supplied by the consumer program per call (or sourced from a
   consumer-owned configuration account); the pull library must not silently
   accept a signer set passed by the data-package author or otherwise sourced
   from the payload.
3. The minimum recommended `maxAge` for production use is documented for both
   modes, with a manipulation analysis covering signer compromise, replay of
   stale packages, and (push mode) signer-set update delays. The pull-mode
   guidance must include the consumer-side responsibility of keeping the
   authorised signer set fresh (since pull bypasses the aggregator's centrally
   administered set).

### Soft Requirements

1. Multi-feed batched verification: amortise calldata and signature recovery
   overhead across multiple feeds in a single instruction (analogous to Pyth's
   Perseus amortisation).
2. Multi-source integration test against the on-chain TWAP tier from RFP-019
   once the TWAP program is available, covering at least one variant per
   RedStone mode: confirm that a consumer reading the RedStone price account
   (push) plus the TWAP price account, and a consumer running public-mode pull
   verification plus reading the TWAP price account, for the same
   `(base_asset, quote_asset)` pair can each apply an example cross-source
   policy (primary plus fallback, divergence cross-check) without the adaptor
   participating in that policy.

### Out of Scope

The following are explicitly excluded from this RFP and addressed elsewhere:

- The on-chain TWAP tier and the canonical oracle price account standard are
  owned by [RFP-019](./RFP-019-twap-oracle.md). This RFP populates the standard,
  it does not define it.
- Adaptors for other off-chain oracles (Pyth, Chainlink, DIA, Chronicle,
  Switchboard, Supra). Future RFPs may add them.
- Pull-mode reads from inside *private* execution. Public-mode pull is delivered
  (see Functionality #9 and the Design Rationale section "Push aggregator and
  public-mode pull"). A private transaction that wants to verify a secp256k1
  signature inline cannot do so without forfeiting batching benefits or breaking
  privacy. Private composability with the feed is via reading the public price
  account that the push aggregator writes to.
- Adding a secp256k1 ECDSA + keccak256 precompile to LEZ. The RISC-V in-program
  path is the deliverable here. A precompile becomes a candidate for a follow-on
  RFP if and only if the cost measurement in this RFP shows the in-program path
  is too expensive for production cadence.
- Price feed composition (combining two or more price accounts whose
  denominations chain together, e.g. computing `LGS/USD = LGS/wBTC × wBTC/USD`).
  RFP-019's canonical standard exposes `base_asset` and `quote_asset` to make
  composition checkable, but neither RFP-019 nor this adaptor specifies or
  implements the composition itself. Composition becomes relevant only once
  token wrapping is defined on LEZ; a future RFP, likely an evolution of this
  one or a dedicated token-wrapping RFP, is expected to specify a canonical
  composition pattern (confidence- interval and staleness rules across legs).
  Until then, consumer protocols that cross denominations are responsible for
  their own composition logic.

## ⚠ Platform Dependencies

### Hard blockers

None at the runtime level. The adaptor builds on the LEZ runtime as it stands
today (RISC-V zkVM on RISC0, public-execution mode, public account storage).
Signature verification runs as in-program code; no new precompile or syscall is
required to deliver the adaptor.

#### Admin authority (RFP-001)

The Functionality requirements specify that an admin authority (per RFP-001,
integrated via the SPEL framework) registers feeds, deregisters them, and
updates each feed's authorised signer set. These admin-gated functions require
the standardised admin authority library from
[RFP-001](./RFP-001-admin-authority-lib.md), currently in development.

### Soft blockers

#### Event emission (LP-0012)

Analytics and monitoring benefit from structured on-chain events for price
updates, feed registrations, and signer-set changes.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is **closed** (delivered).

## 👤 Recommended Team Profile

Team experienced with:

- Oracle or DeFi protocol infrastructure development
- Cryptographic verification (secp256k1 ECDSA recovery, keccak256 hashing,
  calldata parsing, signer-set management)
- LEZ / RISC0 program development; in particular, comfort writing and measuring
  elliptic-curve and hash-function code in RISC-V programs proved by RISC0 (cost
  characterisation experience is a strong signal, since cost measurement is a
  primary deliverable)
- RedStone's data-package format, EVM connector, or Solana connector (any prior
  integration is a strong signal)
- Smart-contract security auditing (signer compromise, replay attacks,
  signer-set update races)
- Operating production oracle or relayer infrastructure under an SLA (uptime
  monitoring, incident response, on-call), since the awarded team runs the live
  feeds through the operating period (see Servicing)

## ⏱ Timeline Expectations

Estimated software delivery duration: **6 to 10 weeks**, followed by an
**operating period running from software readiness through March 2028** (one
year past the planned mainnet launch), during which the awarded team runs the
feeds under the Servicing requirements.

The adaptor has no hard runtime dependencies; it builds on LEZ as it stands
today. The canonical price account standard is a soft dependency on RFP-019 with
a documented fallback. The cost measurement deliverable resolves the open
question of whether in-program ECDSA + keccak256 in RISC0 is fast enough at
production cadence for both the push-mode aggregator and public-mode pull; if
not for either, a follow-on RFP for a secp256k1 precompile becomes the
optimisation path, with this adaptor as the immediate consumer in both modes.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (consumer of price feeds; private swaps consume the RedStone-published prices)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (primary consumer of price feeds)
- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
  (reflexive stablecoin; consumer of external-price feeds for any wrapped-asset
  collateral path)
- [RFP-019 — On-Chain TWAP Oracle](./RFP-019-twap-oracle.md) (defines the
  canonical oracle price account standard)
- [LEZ programs](https://github.com/logos-blockchain/lez-programs): on-chain
  TWAP program reference implementation
- [Appendix: Oracle Ecosystem](../appendix/oracle-ecosystem.md)
- [RedStone Documentation](https://docs.redstone.finance/)
- [RedStone token registry](https://github.com/redstone-finance/redstone-api/blob/main/docs/ALL_SUPPORTED_TOKENS.md)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
