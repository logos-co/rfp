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

Build a RedStone off-chain oracle adaptor for LEZ: a public-mode LEZ program
that verifies RedStone-signed data packages, exposes the resulting prices
through the canonical oracle price account standard defined in
[RFP-019](./RFP-019-twap-oracle.md), and supports day-one delivery of BTC/USD,
ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds. RedStone's data packages are
signed with secp256k1 + keccak256 by its data nodes; verification on LEZ runs
as in-program code inside the RISC-V zkVM (no cross-chain bridge, no Wormhole
dependency). The adaptor uses a push-mode aggregator pattern: a public-mode
program verifies signatures on the write side, stores the result in a public
price account, and consumers (including private-execution programs) read the
slot.

LEZ is RISC0-based, so any signature scheme can be implemented in program
code. Early prototype work on in-program secp256k1 ECDSA verification inside
RISC0 ([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench))
shows the verification is slow enough that **pull-mode reads from inside a
private transaction are not feasible on consumer hardware** (a private
consumer would spend several minutes generating the proof for each read),
absent a RISC0-specific signature-verification accelerator. The push-mode
aggregator pattern is therefore the working assumption: the verifier runs once
per update on the write side in public execution; consumers (public or
private) read the resulting public price account without doing any signature
work. Cost measurement remains a primary deliverable for the public-mode write
side, where amortisation across all downstream reads can make in-program
verification workable; if the measured public-mode cost is unacceptable the
measurement becomes the input to a follow-on RFP that proposes adding a
secp256k1 ECDSA + keccak256 precompile to LEZ.

### Scope

In scope:

- The RedStone off-chain oracle adaptor as a public-mode push aggregator on
  LEZ.
- Day-one BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds, registered
  and exercised on LEZ devnet/testnet.
- Cost measurement of the in-program RISC-V verification path as a primary
  deliverable.

Out of scope at the Overview level (full list under Out of Scope below):

- The on-chain TWAP tier and the canonical oracle price account standard,
  owned by [RFP-019](./RFP-019-twap-oracle.md).
- A Pyth adaptor: depends on Wormhole on LEZ, deferred to a future RFP.
- Pull-mode reads from inside private execution: blocked on either a RISC0
  zkVM circuit-level accelerator (e.g. a future `risc0-ecdsa` extension that
  lowers in-circuit verification cost) or a different upstream signature
  scheme that admits acceptable in-circuit cost on RISC0; neither exists
  today. A LEZ secp256k1 + keccak256 *precompile* (a runtime host function) is
  the subject of a possible cost-conditional public-mode follow-on RFP; that
  precompile lives outside the ZK proof boundary, so it lowers public-mode
  write-side cost but does *not* unblock private-execution pull (a host
  function cannot be called from inside the privacy circuit). An alternative
  private-mode-friendly signature scheme (with the corresponding upstream
  publisher work) is the subject of a possible separate follow-on RFP.
  Whether either follow-on is warranted depends on consumer-protocol demand
  for private-execution pull, which is not yet established: some consumer
  protocols (notably the reflexive stablecoin in
  [RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)) already constrain
  specific actions to public transactions for their own design reasons, so
  the capability is worth reaching for only if a downstream consumer actually
  needs it.

## 🔥 Why This Matters

Private DeFi is what LEZ is positioned to support, and reliable USD reference
prices for privacy collateral, in particular Monero (XMR) and Zcash (ZEC), are
a necessary step. The reflexive stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), the privacy-preserving
DEX ([RFP-004](./RFP-004-privacy-preserving-dex.md)), wrapped privacy assets,
and other cross-chain primitives all need those reference prices to function.

The on-chain TWAP tier in [RFP-019](./RFP-019-twap-oracle.md) is not sufficient
on its own for the day-one asset list. TWAP security scales linearly with pool
depth: on a new chain where liquidity is thin, a validator controlling two
consecutive blocks can manipulate the accumulator at a cost roughly equal to
the round-trip swap fees and price impact, which on a $1M pool is cheap (see
[Appendix: TWAP Manipulation Vectors](../appendix/oracle-ecosystem.md)).
This applies even to majors like BTC/USD, ETH/USD, and SOL/USD: those pairs
will exist as pools on LEZ, but at chain launch their pool depth will not yet
be sufficient for TWAP to stand alone. More structurally, TWAP only produces a
price for pairs that exist as pools on LEZ; XMR/USD and ZEC/USD don't, because
XMR and ZEC aren't natively on LEZ. An off-chain feed is the only way to get
those prices on chain at all, and pairing it with TWAP for the pairs where
TWAP does work is the production norm for layered oracle defence.

Across the surveyed off-chain oracle providers, RedStone is the only one that
combines: support for both XMR and ZEC in its public token registry, a
portable connector pattern (single secp256k1 ECDSA + keccak256 verification
path that works the same on every host chain), no cross-chain bridge
requirement, and a self-serve deployment path that does not require an
oracle-team business engagement. Pyth covers both feeds and adds higher
publisher counts and confidence intervals, but is gated on Wormhole
integration on LEZ; it should land as a fast-follow in a future RFP.
Chainlink is permissioned and not self-serve. DIA Lumina is permissionless but
requires bespoke per-chain deployment. See
[Appendix: Oracle Ecosystem, Privacy-Asset Feed Availability](../appendix/oracle-ecosystem.md)
for the full coverage matrix.

The combination of "private DeFi needs XMR and ZEC" and "RedStone is the only
path that is self-serve on LEZ today" makes this the priority off-chain oracle
integration for LEZ.

### A building block in a layered oracle stack

Neither an off-chain feed nor an on-chain TWAP is a complete oracle on its
own; both have known failure modes and the production norm in DeFi is to layer
them. This RFP delivers the off-chain adaptor as a swappable building block in
that layered stack: production-grade code on its own terms, paired with the
TWAP tier from [RFP-019](./RFP-019-twap-oracle.md) on the consumer side.
Consuming protocols (the reflexive stablecoin in
[RFP-013](./RFP-013-reflexive-stablecoin-protocol.md), the lending market in
[RFP-008](./RFP-008-lending-borrowing-protocol.md), the DEX in
[RFP-004](./RFP-004-privacy-preserving-dex.md)) compose these pieces according
to their own production-security choices, with the canonical price account
standard from RFP-019 keeping swap-out cheap if those choices change later.

## 🏗 Design Rationale

### Public-mode aggregator with private-account composability

The adaptor runs as a public-mode LEZ program with no confidential state.
Signature verification, data-package decoding, and price publication are all
visible to any caller. Any LEZ dapp can read the same canonical price.

This shape is determined by where signature verification can run on LEZ. LEZ
is a RISC-V zkVM built on RISC0; any code that runs inside a private
transaction has to be expressible inside the RISC-V zkVM circuit, so a private
transaction that wants to verify a secp256k1 ECDSA signature has two options,
both unappealing: verify the signature inside the privacy circuit (forfeits
the batching benefits that make ZK proof amortisation work; RISC0
elliptic-curve performance for this primitive is currently unmeasured), or
place the signature in the transaction journal where it is publicly disclosed
(breaks the privacy of the transaction). Neither option preserves both
efficiency and privacy.

The adaptor therefore runs the verifier in a public-mode aggregator:
signatures are recovered once per update on the write side, and the verified
price plus timestamp are stored in a public price account. Private-execution
programs compose with the price by reading the public account, not by
carrying signed payloads inline. Cost is paid once per update and amortises
across all downstream reads, public and private. Confidential execution is
reserved for application-layer protocols that consume oracle prices (for
example, private DEX swaps in
[RFP-004](./RFP-004-privacy-preserving-dex.md)); the price feed itself stays
public.

Pull-mode reads (where a public consumer transaction carries a signed payload
and verifies inline) remain technically possible on LEZ inside public
execution, but are out of scope for this RFP because they don't extend to
private execution and because the push-mode aggregator gives strictly better
cost amortisation for the LEZ DeFi consumer set. They can be revisited in a
follow-on once measured cost data is in.

See [Appendix: Oracle Ecosystem, Implications for LEZ](../appendix/oracle-ecosystem.md#implications-for-lez)
for the full four-shape analysis (trusted re-signer, FROST-BIP340 federation,
DLC-oracle extension, and the cost-conditional precompile path) that this
section condenses.

A LEZ-specific freshness pattern follows from the public / private execution
split. A user who needs a price fresher than the heartbeat's last update can
submit a public transaction that pushes a fresh signed payload to the
aggregator account, then submit a private transaction immediately after that
reads the just-updated public price. Verification cost is paid in the public
path (cheaper, especially under the precompile follow-on); the private
transaction does no signature work. This recovers pull mode's "fresh at
transaction time" property for private consumers without paying the
in-circuit cost in the privacy proof. The adaptor program already
accommodates this: any caller can submit a valid signed payload and the
program writes if signatures and timestamps check out.

The same mechanism enables a **consumer-pays push variant** that does not
require a dedicated relayer at all. Users push when they need a fresh price;
idle periods incur zero update cost; the aggregator only advances when
someone needs it. This is operationally pull (consumer-pays, on-demand) but
structurally push (the program owns the public price account that downstream
private consumers read from). Whether to run a heartbeat relayer in addition
(RedStone's own pusher, a sovereign relayer, or neither) is a deployment-time
choice: a heartbeat keeps the slot[]() warm for read-only consumers;
consumer-pays push keeps the cost model strictly proportional to demand. Both
can coexist; the program logic does not distinguish between them.

The two-transaction split (public push, then private read) is distinct from
the rejected "put the signature in the journal" option. The signature is
carried only in the public push; the private transaction reads the resulting
public price account by address with no upstream signature in its calldata or
journal, so the private transaction's contents (assets, counterparty, amount)
stay private. This fits the adaptor's existing write path: the program
already accepts signed payloads from any caller and writes a public price
account, regardless of whether the caller is a relayer or an end user. There
is a residual linkability risk that the consumer needs to handle on its side,
not the adaptor's: an observer can correlate a public push from wallet X at
time T with a private transaction at time T plus epsilon and infer that the
same actor is consuming the just-pushed price. Mitigations (separate funding
wallet for the push, timing decorrelation, reliance on a heartbeat to mask
single-purpose pushes) are consumer-side production-security choices. The
privacy story is strictly better than journal-disclosed signatures (the
private transaction's body stays private) but not equivalent to a
heartbeat-only push model.

### RISC-V verification path and the precompile question

This RFP implements signature verification in RISC-V program code, running
inside RISC0. There is no host primitive to call: the recovery is an
in-program ECDSA + keccak256 path written against existing Rust crates (k256 /
sha3 / equivalents) and proved by RISC0 along with the rest of the program.

This is the central technical bet of the RFP. Early prototype work on
in-program secp256k1 ECDSA verification inside RISC0
([`fryorcraken/lez-signature-bench`](https://github.com/fryorcraken/lez-signature-bench))
is already enough to flag that the naive in-circuit path is slow on consumer
hardware: a private consumer attempting pull-mode verification would spend
several minutes generating the proof for each read. That rules out
private-execution pull mode in any practical sense for the in-program path,
absent a RISC0-specific signature-verification accelerator (e.g. a future
`risc0-ecdsa` extension or a secp256k1 precompile wired into the zkVM proving
system itself). The first concrete deliverable of this RFP refines this
picture for the public-mode write side: implement the verifier in RISC-V, run
it on LEZ, document the cost (compute units, proof time, proof size,
per-update bytes) for both the per-signature recovery and the full 3-of-N
aggregator write, and characterise where the public-mode cost lands relative
to the production-cadence budget. Public-mode cost is the cost that matters
for shipping the adaptor: it amortises across all downstream reads.

Two outcomes are possible from that measurement:

1. **Measured cost is acceptable for the push-mode aggregator.** The adaptor
   ships on the runtime as it stands. The aggregator's update cadence
   amortises the per-update cost across all downstream reads. No runtime
   change required.
2. **Measured cost is unacceptable.** The measurement becomes the input to a
   follow-on RFP that proposes adding a secp256k1 ECDSA + keccak256 precompile
   to LEZ for use by public-execution programs. A precompile lives outside the
   ZK proof boundary and is invoked as native validator code, so the cost goes
   from "ZK-proven elliptic-curve operations" to "native ECDSA recovery +
   keccak", which is the cost profile RedStone's existing connectors assume on
   every other chain. The precompile is an optimisation path conditional on
   the measurement, not a precondition for this RFP.

The applicant should therefore design the verification path so that swapping
in a precompile in a later release is a localised change (a single trait
implementation or syscall wrapper), not a restructuring of the program.

The precompile path addresses public-mode write-side cost only;
private-execution pull mode is foreclosed under it for the same structural
reason it is foreclosed under the in-program path (a precompile lives outside
the ZK proof boundary, so it is not callable from inside a private
transaction). If a signature scheme exists, or can be selected, that yields
acceptable in-circuit cost on RISC0 (a RISC0-friendly hash and curve
combination, or a different signature primitive that admits cheaper
in-circuit verification), pull-mode reads from inside private execution
become reachable and the public-mode aggregator becomes one option among
several rather than the only viable shape. Identifying or building such a
scheme, and either modifying an existing oracle network to publish in it or
standing up a new publisher set that does, is the subject of a possible
separate follow-on RFP. It is independent of the public-mode precompile
question and out of scope for this RFP. Whether such a follow-on is worth
pursuing depends on whether any consumer protocol actually needs
private-execution pull, which is not yet confirmed: consumer protocols may
already have design reasons to keep specific actions in public transactions,
with the reflexive stablecoin in
[RFP-013](./RFP-013-reflexive-stablecoin-protocol.md) as one concrete example.

### Why RedStone first

Three reasons specific to LEZ's constraints:

1. **Privacy-asset coverage with no bridge.** Both XMR and ZEC are in
   RedStone's public token registry, and the RedStone connector pattern is
   fully self-serve: deployment does not require an oracle-team engagement, a
   bridge, or a per-chain registration step. As far as we can see from the
   public SDK
   ([`@redstone-finance/sdk` on npm](https://www.npmjs.com/package/@redstone-finance/sdk))
   and the
   [RedStone Pull docs](https://docs.redstone.finance/docs/dapps/redstone-pull/),
   consuming the live data-package gateways requires no account, API key, or
   signup at the technical level; use is governed by the public
   [RedStone Terms of Use](https://redstone.finance/terms-of-use) (acceptance
   implied by use). Whoever uses or runs the resulting software is the party
   bound by those Terms; see "Operator-side T&C considerations" below. LEZ
   can deploy and exercise these feeds without touching any external
   infrastructure. Pyth and DIA both cover the same assets but require either
   Wormhole (Pyth) or bespoke per-chain deployment (DIA Lumina) before they
   work on a new chain.
2. **Single verification primitive, no bridge.** RedStone's signature scheme
   is plain m-of-N secp256k1 ECDSA over keccak256 (typically 3-of-N).
   Verification on LEZ is in-program ECDSA recovery and keccak256 hashing
   inside RISC0; the cost profile is the open variable this RFP measures (see
   "RISC-V verification path and the precompile question"). Pyth's full
   13-of-19 Wormhole VAA verification is heavier in two ways: it adds a
   Merkle proof on top of more signatures, and it presupposes a Wormhole
   guardian-set tracking program on LEZ that does not yet exist. RedStone has
   neither cost.
3. **Independent of LEZ's external integration timeline.** Choosing RedStone
   first decouples the oracle layer from when Wormhole on LEZ is decided.
   Pyth then fast-follows in a future RFP, contributing higher publisher
   counts (especially the roughly 80+ on XMR/USD versus RedStone's smaller
   per-feed roster) and confidence intervals that RedStone does not natively
   expose.

See
[Appendix: Oracle Ecosystem, Signature Verification Schemes](../appendix/oracle-ecosystem.md)
for the full per-scheme analysis and citations.

### Conformance to the canonical price account standard

The canonical oracle price account standard is owned by RFP-019 (see "LEZ
oracle data standard" in that RFP's Design Rationale). The RedStone adaptor
populates the same struct as the on-chain TWAP source: `base_asset`,
`quote_asset`, price, timestamp, source identifier, and confidence interval
(zero, since RedStone does not publish one). Consuming protocols query a
single data layout and remain agnostic to whether the price came from TWAP,
RedStone, or any future provider; cross-source policy lives in the consumer
per RFP-019, Design Rationale ("Multi-source coexistence").

If RFP-019 has not yet shipped the canonical struct when this RFP is
delivered, the team must define a forward-compatible minimal struct using
append-friendly account-data conventions, so that a later RFP-019 release can
extend the struct without breaking consumers.

### HTTPS data path: centralisation and censorship

Signed RedStone data packages are fetched from the RedStone Data Distribution
Layer (DDL) over **HTTPS, not P2P**. The default gateway URLs are DNS-resolved
endpoints under `*.redstone.finance` and `*.redstone.vip` (see the SDK source
at `packages/sdk/src/data-services-urls.ts` and the appendix section
"Infrastructure Requirements for External Oracles on LEZ"). This applies
whether the data reaches LEZ via the relayer (push) or inline as transaction
calldata (pull). Two consequences follow:

- **Censorship surface.** A gateway operator (and its upstream network) can
  withhold responses to specific clients, geographies, or asset IDs. On-chain
  signature verification protects against falsified data, not against
  withheld data: a censored consumer cannot obtain a signed payload, and any
  transaction depending on it reverts.
- **Liveness dependency.** DNS, TLS validity, and HTTPS availability at
  RedStone's hosts (AWS, GCP) all become LEZ oracle liveness dependencies.
  Regional outages at these providers can take the feed offline.
- **Privacy leak in pull mode.** Pull-mode consumers fetch the signed data
  package directly from the RedStone DDL gateway before submitting it inline
  with their on-chain transaction. This reveals the consumer's IP (and
  timing, and which asset pair was requested) to the gateway operator
  *before* the on-chain action, even when the on-chain action itself runs in
  private execution. The push-aggregator shape this RFP commits to confines
  this exposure to the relayer operator: the relayer's IP is visible to
  RedStone, but downstream consumers read from the public price account on
  LEZ with no gateway contact. This is an additional reason to prefer the
  push-aggregator pattern over pull mode on LEZ, separate from the
  in-circuit cost argument.

Implementers must document, as part of the relayer operator journey, the
mitigations the operator chooses: multiple parallel gateway queries (SDK
default), the use of RedStone private gateways via the
`OVERRIDE_DIRECT_CACHE_SERVICE_URLS` mechanism where available, and
recent-price fallback policy if no gateway responds within the operator's
tolerance. None of these mitigations eliminate the structural centralisation;
they reduce specific failure modes.

This caveat is inherent to RedStone's design and to every oracle in the
survey that uses an HTTPS portal (Pyth Hermes has the same property). It is
not an objection to RedStone; it is the trade-off LEZ accepts by adopting any
HTTPS-served signed-data oracle, and must be communicated to consumer
protocols so they can size their liveness assumptions accordingly.

### Operator-side T&C considerations

This RFP scopes the delivery of *software* (the adaptor program, the relayer
module, the SDK and consumer-pattern documentation). It does not itself
execute any data-fetch request against RedStone's gateways. The party that
uses or runs the resulting software (in particular the relayer that fetches
data packages from `*.redstone.finance` and `*.redstone.vip`) is the party
bound by the
[RedStone Terms of Use](https://redstone.finance/terms-of-use) and must abide
by them.

The Terms restrict commercial use and do not describe a self-serve
application process. Contact paths for a commercial agreement (current as of
this writing, may move):

- `redstone.finance/contact` (general partnership / integration enquiry
  form).
- `redstone.finance/institutional` ("Contact Institutional Sales" form,
  oriented at SLAs, custom feeds, and paid partnerships).
- `dev@redstone.finance` (the only email surfaced in the Terms; appropriate
  for general enquiries).

This RFP does not require the implementer to obtain a commercial agreement;
the implementer's deliverable is software. The implementer's documentation
**must**, however, surface the Terms of Use to relayer operators as part of
the operator journey (see Functionality #7 and Supportability), so any
operator deploying the relayer is on notice that they are the party bound by
the Terms.

### Fee structure

In the push-aggregator shape this RFP commits to, the on-chain verification
cost is paid once per update by whoever submits the signed data package to
the aggregator, and is amortised across all downstream reads (public and
private). RedStone itself does not publish prices on-chain; a relayer fetches
the signed packages from the RedStone gateway and pushes them, so "whoever
pays for an update" in practice means whoever runs (or pays for) the relayer.
The adaptor does not need to fund a dedicated node operator pool.

Beyond that structural point, this RFP does not prescribe a fee model.
Downstream users of the adaptor (consuming protocols, relayer operators,
deployers) are free to handle fees in whatever way fits their product:
subsidised by the consuming protocol, charged per read, charged per update,
routed to a treasury, burned, or left at zero. The adaptor program itself
should not bake in policy that forecloses these choices.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a public-mode LEZ program (push-mode aggregator) that accepts
   signed RedStone data packages, recovers each signer's public key via
   in-program secp256k1 ECDSA recovery (with keccak256 hashing) running
   inside the RISC-V zkVM, and verifies that the recovered public keys match
   the configured set of authorised RedStone data nodes for the requested
   feed. Structure the verification path so that swapping the in-program
   recovery for a future host primitive (precompile or syscall) is a
   localised change.
2. Verify the M-of-N signer threshold for each feed (configurable at
   registration; default 3-of-N consistent with RedStone's reference
   parameters) and reject any data package that does not meet the threshold.
3. Decode the RedStone data package format (asset identifier, value,
   timestamp, signer set) and reject any package whose timestamp is older
   than a configurable `maxAge`, whose value is zero, negative, or otherwise
   invalid, or whose asset identifier does not match the registered feed.
4. Publish the verified price into a canonical oracle price account
   conforming to the standard defined in RFP-019. The adaptor must populate
   `base_asset`, `quote_asset`, price, timestamp, source identifier (a
   constant identifying RedStone), and confidence interval (zero; RedStone
   does not publish confidence intervals in its standard data packages).
5. The adaptor program owner can register new RedStone feeds (by asset
   identifier, M-of-N threshold, and authorised signer set), update an
   existing feed's signer set on RedStone roster changes, and deregister
   feeds.
6. BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feeds must be registered
   and exercised on LEZ devnet/testnet as part of the deliverable.
7. **Relayer module.** Provide a relayer service that fetches signed RedStone
   data packages from the RedStone Data Distribution Layer gateways (see
   Appendix, "Infrastructure Requirements for External Oracles on LEZ") and
   submits them to the LEZ adaptor. The relayer must be implemented as a
   **Logos module accompanied by a Logos Core headless CLI/daemon** (same
   packaging model as the liquidator bot in RFP-008 Functionality #13), so
   that operators (RedStone, the Logos ecosystem, or consuming protocols)
   can run it as a standalone long-running process without requiring a
   user-facing app. The daemon must support: configurable `dataServiceId`
   (default `redstone-primary-prod`), configurable feed list and update
   triggers per feed (heartbeat interval and deviation threshold), parallel
   querying of multiple DDL gateways with first-success selection, retry and
   back-off on transient gateway or LEZ errors, structured logging of
   submitted updates and rejections (with the on-chain rejection reason
   where available), wallet-balance monitoring, and a clean shutdown path.
   Document the operator journey end-to-end: install, configure, run,
   monitor. The operator documentation must explicitly cover the HTTPS
   data-path centralisation discussed in Design Rationale ("HTTPS data path:
   centralisation and censorship"): which gateways are queried, what the
   operator's policy is when none respond, and how the operator detects
   selective censorship. The operator documentation must also surface the
   [RedStone Terms of Use](https://redstone.finance/terms-of-use)
   commercial-use clause discussed in Design Rationale ("Operator-side T&C
   considerations"), so any operator deploying the relayer is on notice
   that fee-charging operation may require a separate commercial agreement
   with RedStone.

#### Usability

1. Provide an SDK that can be used to build Logos modules for submitting
   RedStone data packages and reading verified prices from the canonical
   price account.
2. Provide a Logos mini-app GUI (off-chain feed dashboard) with local build
   instructions, downloadable assets, and loadable in Logos app (Basecamp)
   via git repo. The dashboard must display: live prices for each registered
   feed, the configured signer set, the current M-of-N threshold, the latest
   data-package timestamp, and the staleness of each feed.
3. Provide a CLI that covers core functionality: submit a data package,
   query the verified price, register and deregister feeds, update signer
   sets.
4. Provide an IDL for the adaptor program and the canonical oracle price
   account standard (re-exported from RFP-019, not forked), using the
   [SPEL framework](https://github.com/logos-co/spel).
5. Return clear, actionable error messages for all failure modes: stale data
   package, signer-threshold not met, signer not in authorised set, asset
   identifier mismatch (`base_asset` or `quote_asset` does not match the
   registered feed), malformed package, invalid signature, zero or negative
   price.
6. Provide a **reference consumer program**: a minimal LEZ program (or
   equivalently a documented program-side code snippet plus tests) that
   demonstrates the recommended consumer-side integration pattern for
   reading the canonical price account populated by this adaptor. The
   reference must show: verifying the `(base_asset, quote_asset)` pair
   matches the consumer's expectation, reading price and timestamp from the
   account, rejecting prices older than the consumer's chosen `maxAge`, and
   the recommended response when a price is unavailable (typically: refuse
   the action, do not fall back to an unsafe default). Cross-source policy
   (combining the RedStone feed with an on-chain TWAP or another external
   source) is the consumer protocol's responsibility per RFP-019, Design
   Rationale ("Multi-source coexistence"); this reference must not bundle a
   divergence policy. This is a guidance artefact for downstream consumer
   protocols (RFP-008, RFP-013, RFP-004), not a production product on its
   own.

#### Reliability

1. A price read is read-only and never modifies adaptor state.
2. Feed registration is atomic: partial failure leaves existing registrations
   intact.
3. Signature verification is deterministic: given the same data package and
   signer set, the verification result is the same.

#### Performance

1. End-to-end signature verification and price publication for a single
   3-of-N RedStone data package must complete within a single LEZ public
   transaction at the per-transaction compute and proof budget in force on
   LEZ at delivery time.
2. Cost measurement is a primary deliverable, not a side report. The
   applicant must measure and document, for the RISC-V in-program
   verification path: per-signer ECDSA recovery cost (compute units, RISC0
   proof time, RISC0 proof size), keccak256 hashing cost, package decoding
   cost, signer-set membership check, canonical price account write, and
   feed registration. Numbers must be reproducible from the test suite.
3. Document the cost delta between the in-program path and a hypothetical
   native ECDSA + keccak256 precompile, using existing per-chain reference
   points (for example, the RedStone EVM end-to-end gas range of 50K to
   100K, and the per-recovery cost profile on chains that expose a native
   primitive). The delta informs whether a follow-on precompile RFP is
   warranted.

#### Supportability

1. The adaptor program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone
   mode) and are included in CI; CI must be green on the default branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test. The test suite must
   include: valid signature acceptance, invalid signature rejection,
   signer-threshold enforcement (M-of-N, including boundary cases),
   stale-package rejection (`maxAge`), asset-identifier mismatch rejection,
   zero or negative price rejection, and feed registration / signer-set
   update transitions.
4. A README documents end-to-end usage: deployment steps, program addresses,
   initial BTC/USD, ETH/USD, SOL/USD, XMR/USD, and ZEC/USD feed
   registrations, and step-by-step instructions for submitting data packages
   and querying prices via CLI and mini-app.
5. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for submitting
   RedStone data packages and reading verified prices, **plus a "Recommended
   Consumer Pattern" section** that walks a downstream protocol developer
   through the reference consumer program from Usability #6:
   `(base_asset, quote_asset)` verification, staleness handling, behaviour
   when a price is unavailable, and an example multi-source policy pairing
   the RedStone feed with the on-chain TWAP tier from RFP-019. The doc must
   state explicitly that cross-source policy is owned by the consumer
   protocol, not by this adaptor.
6. Submit a [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
7. Provide Figma designs or equivalent for the mini-app GUI (off-chain feed
   dashboard).

#### + Adaptor Security

1. The adaptor must reject any data package whose recovered signer is not in
   the configured authorised signer set for the requested feed.
2. The signer set must be updatable only by the program owner; the update
   path itself must be tested.
3. The minimum recommended `maxAge` for production use is documented, with a
   manipulation analysis covering signer compromise, replay of stale
   packages, and signer-set update delays.

### Soft Requirements

1. Multi-feed batched verification: amortise calldata and signature recovery
   overhead across multiple feeds in a single instruction (analogous to
   Pyth's Perseus amortisation).
2. Multi-source integration test against the on-chain TWAP tier from RFP-019
   once the TWAP program is available: confirm that a consumer reading both
   the RedStone price account and the TWAP price account for the same
   `(base_asset, quote_asset)` pair can apply an example cross-source policy
   (primary plus fallback, divergence cross-check) without the adaptor
   participating in that policy.

### Out of Scope

The following are explicitly excluded from this RFP and addressed elsewhere:

- The on-chain TWAP tier and the canonical oracle price account standard are
  owned by [RFP-019](./RFP-019-twap-oracle.md). This RFP populates the
  standard, it does not define it.
- A Pyth adaptor. Pyth depends on Wormhole on LEZ and is deferred to a future
  RFP. Higher publisher counts and confidence intervals (which RedStone does
  not natively expose) come with that adaptor.
- Adaptors for other off-chain oracles (Chainlink, DIA, Chronicle,
  Switchboard, Supra). None of these match the combination of privacy-asset
  coverage, single-primitive verification, and bridge independence that
  motivates this RFP. Future RFPs may add them.
- Pull-mode reads from inside private execution. A private transaction that
  wants to verify a secp256k1 signature inline cannot do so without
  forfeiting batching benefits or breaking privacy (see Design Rationale).
  Private composability is via reading the public price account that the
  push-mode aggregator writes to.
- Adding a secp256k1 ECDSA + keccak256 precompile to LEZ. The RISC-V
  in-program path is the deliverable here. A precompile becomes a candidate
  for a follow-on RFP if and only if the cost measurement in this RFP shows
  the in-program path is too expensive for production cadence.
- Switching the upstream signature scheme to one that yields acceptable
  in-circuit cost for private execution on RISC0 (RISC0-friendly hash and
  curve, or a different signature primitive entirely). If such a scheme
  exists or can be selected, modifying an existing oracle network to publish
  in it, or standing up a new publisher set that does, would unlock
  pull-mode reads from inside private transactions and is the subject of a
  possible separate follow-on RFP. It is independent of the public-mode
  precompile question above and is not a deliverable of this RFP. The
  follow-on is itself contingent on consumer-protocol demand for
  private-execution pull mode, which is not yet confirmed: parts of the
  reflexive stablecoin design in
  [RFP-013](./RFP-013-reflexive-stablecoin-protocol.md), for instance,
  already constrain specific actions to public transactions, so the
  capability is worth pursuing only if a downstream consumer actually
  requires it.
- Price feed composition (combining two or more price accounts whose
  denominations chain together, e.g. computing
  `LGS/USD = LGS/wBTC × wBTC/USD`). RFP-019's canonical standard exposes
  `base_asset` and `quote_asset` to make composition checkable, but neither
  RFP-019 nor this adaptor specifies or implements the composition itself.
  Composition becomes relevant only once token wrapping is defined on LEZ; a
  future RFP, likely an evolution of this one or a dedicated token-wrapping
  RFP, is expected to specify a canonical composition pattern (confidence-
  interval and staleness rules across legs). Until then, consumer protocols
  that cross denominations are responsible for their own composition logic.

## ⚠ Platform Dependencies

### Hard blockers

None at the runtime level. The adaptor builds on the LEZ runtime as it stands
today (RISC-V zkVM on RISC0, public-execution mode, public account storage).
Signature verification runs as in-program code; no new precompile or syscall
is required to deliver the adaptor.

### Cost-conditional follow-on (not a blocker for this RFP)

#### secp256k1 ECDSA + keccak256 precompile in public-execution mode

If the cost measurement deliverable shows that in-program ECDSA recovery and
keccak256 hashing in RISC0 are too expensive for the push-mode aggregator's
production cadence, a follow-on RFP can propose adding a precompile (or
accelerated host function) to LEZ for use by public-execution programs. That
RFP would substitute for the in-program verification path in this adaptor via
the localised swap-out described in the Functionality requirements. The
precompile would be public-mode only; private execution paths are unaffected
because they do not call this primitive.

The LEZ runtime team has noted that supporting a secp256k1 primitive raises a
broader set of design questions (nullifier tracking for replay,
privacy-circuit branching to support Ethereum-signed private accounts,
identifier-flow / wallet implications) that are not blockers for the narrow
oracle use of the precompile but should be acknowledged. Those questions can
be scoped out of the follow-on or addressed in a separate runtime RFP,
depending on appetite.

### Soft blockers

#### RFP-019 (canonical oracle price account standard)

This RFP populates the canonical price account standard defined in
[RFP-019](./RFP-019-twap-oracle.md). If RFP-019 has not landed when this RFP
is delivered, the applicant defines a forward-compatible minimal struct (see
Design Rationale).

#### Event emission (LP-0012)

Analytics and monitoring benefit from structured on-chain events for price
updates, feed registrations, and signer-set changes.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

## 👤 Recommended Team Profile

Team experienced with:

- Oracle or DeFi protocol infrastructure development
- Cryptographic verification (secp256k1 ECDSA recovery, keccak256 hashing,
  calldata parsing, signer-set management)
- LEZ / RISC0 program development; in particular, comfort writing and
  measuring elliptic-curve and hash-function code in RISC-V programs proved
  by RISC0 (cost characterisation experience is a strong signal, since cost
  measurement is a primary deliverable)
- RedStone's data-package format, EVM connector, or Solana connector (any
  prior integration is a strong signal)
- Smart-contract security auditing (signer compromise, replay attacks,
  signer-set update races)

## ⏱ Timeline Expectations

Estimated duration: **6 to 10 weeks**.

The adaptor has no hard runtime dependencies; it builds on LEZ as it stands
today. The canonical price account standard is a soft dependency on RFP-019
with a documented fallback. The cost measurement deliverable resolves the
open question of whether in-program ECDSA + keccak256 in RISC0 is fast enough
for the push-mode aggregator at production cadence; if not, a follow-on RFP
for a secp256k1 precompile becomes the optimisation path, with this adaptor
as the immediate consumer.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (consumer of price feeds; private swaps consume the RedStone-published
  prices)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (primary consumer of price feeds)
- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
  (reflexive stablecoin; consumer of external-price feeds for any
  wrapped-asset collateral path)
- [RFP-019 — On-Chain TWAP Oracle](./RFP-019-twap-oracle.md) (defines the
  canonical oracle price account standard)
- [Appendix: Oracle Ecosystem](../appendix/oracle-ecosystem.md)
- [RedStone Documentation](https://docs.redstone.finance/)
- [RedStone token registry](https://github.com/redstone-finance/redstone-api/blob/main/docs/ALL_SUPPORTED_TOKENS.md)


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please
use **Discussions**.
