---
id: RFP-021
title: LEZ Indexer FFI
tier: L
status: open
category: Developer Tooling & Infrastructure
dependencies: []
---

# RFP-021 — LEZ Indexer FFI

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

Build the read API for LEZ: the set of functions an integrator needs to follow
the chain, track deposits, and confirm transactions, delivered as exported
functions on the LEZ indexer FFI and surfaced through
[`lez_indexer_module`](https://github.com/logos-blockchain/lez-indexer-module).
This is the equivalent of a node API, in the sense that the `eth` namespace is
on Ethereum's JSON-RPC: it answers what happened and what the current state is,
and it signs nothing.

The FFI is where the LEZ read surface is bounded. A capability the indexer holds
but the FFI does not export cannot reach an application, and adding it to the
module alone achieves nothing. See
[Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md) for the
as-built inventory of every layer.

Of the integrators listed below, the centralised exchange is used as the
reference profile for the requirements: read chain state, track deposits
credited to accounts it controls, and confirm that a transaction reached a level
of certainty it is willing to act on. It is the strictest reader of the set, so
a surface that satisfies it satisfies the others, and it is the profile whose
absence is most visible, since a chain no exchange will list is a chain most
users cannot reach.

### The six deliverables, and which one this is

The six deliverables exist to make LEZ integrable by the parties that have to
integrate a chain before it is usable in practice: wallets, centralised
exchanges, custodians, payment gateways, data and price aggregators, node and
RPC providers, fiat on and off ramps, bridges, and tax and accounting providers.
Each acts on the chain from outside it, holding accounts, watching for what
arrives, signing what it sends, and reconciling against its own books. Each
stops at the first capability that is missing.

A CEX backend and a self-custodial mobile wallet have different architectures
and are written in different languages, which is why this RFP suite takes a
composable approach. The first two deliverables define surfaces; the rest
consume them.

1. **The FFI API for the LEZ indexer.** *This RFP*. The equivalent of a node
   API, in the sense the `eth` namespace is on Ethereum's JSON-RPC. The indexer
   is expected to run as a node, in the shape an RPC provider runs one.
2. **The FFI API for the LEZ wallet**: key handling, derivation, and signing.
   Inside Basecamp it runs as a binary, the `lez_core` module
   ([`logos-execution-zone-module`](https://github.com/logos-blockchain/logos-execution-zone-module));
   outside it, the intent is to ship a library per language from one Rust core,
   as [`bdk-ffi`](https://github.com/bitcoindevkit/bdk-ffi) does (item 4).
3. **The JSON-RPC proxy module**: a module that exposes the FFI API over
   JSON-RPC. It covers both the wallet and the indexer, the indexer being the
   more critical half.
4. **The wallet SDK**: a library per language over the wallet FFI, BDK-shaped
   rather than a client for a wire protocol.
5. **The indexer SDK**: a library for reaching the indexer's JSON-RPC surface.
   The wallet SDK may use it to reach a running indexer over JSON-RPC, so Rust
   is required; other languages follow demand.
6. **Further transport proxy modules** beyond JSON-RPC, such as gRPC, GraphQL,
   and a Mesh or Rosetta adapter.

This RFP defines the indexer read surface only (1). Wallet and key management,
transaction construction and signing, transaction submission, the JSON-RPC
transport, the further transport bindings, and the language SDKs are out of
scope and will be defined in separate RFPs.

Two consequences of that arrangement bear on this RFP. The indexer FFI is
consumed both directly, by anything linking it in process, and indirectly,
through the JSON-RPC proxy and the indexer SDK, so its surface has to survive
projection onto a wire protocol rather than assuming an in-process caller. And
because the wallet SDK may reach the indexer through that same JSON-RPC path
rather than through the FFI, the two surfaces must express the same semantics.

### The sequencer is not a bridging target

The LEZ sequencer exposes a JSON-RPC interface that the wallet currently uses
for account queries. That arrangement is explicitly transitional: the
sequencer's own source marks those methods for removal "after wallet starts
using indexer for this type of queries"
([`lez/sequencer/service/rpc/src/lib.rs:46-47`](https://github.com/logos-blockchain/logos-execution-zone/blob/47eba256479f6f785acbd138834340703cd03401/lez/sequencer/service/rpc/src/lib.rs#L46-L47)).

We further assume that indexer to sequencer communication will move from
JSON-RPC to a libp2p mempool, which is why the sequencer API is not addressed by
this RFP or by the others in this set.

## 🔥 Why This Matters

LEZ is adopted when existing projects integrate it. To increase the value a
project sees in integrating LEZ, we need to decrease the upfront cost by
providing the software they need.

So the requirements below are set by what those integrators already rely on
elsewhere. Eight established chains were surveyed for that purpose. LEZ does not
currently meet several of the norms: there is no simulation, present on six of
the eight; no per-transaction effects; no confirmation level; and no
subscription. See
[Appendix: Blockchain API and SDK Ecosystem, section 4](../appendix/blockchain-api-sdk-ecosystem.md#4-gap-summary).

Most of the missing capability already exists one layer below the FFI, as the
readiness markers below record, so the work is largely exposure rather than
invention.

## 🏗 Design Rationale

### Effects and simulation are one deliverable

The effects query and the simulation query share the executor, the state, and
the diff type
(`StateDiff { signer_account_ids, public_diff, new_commitments, new_nullifiers, program }`).
Reading the stored diff for a transaction already in a block is an effects API;
running the same computation against a caller-supplied transaction, without
applying it, is a simulation API. Specifying them together avoids two designs of
the same thing. The caveat a proposal must carry: the indexer simulates against
finalised state rather than a live tip, a property to document rather than a
defect to fix.

### A status API is not built on `bedrock_status`

A per-transaction status is derived from whether the transaction is present in
the store, which block holds it, and where that block sits relative to the
indexed tip. It is not derived from `Block.bedrock_status`, which the indexer
overwrites to `Finalized` unconditionally before persisting
(`lez/indexer/core/src/block_store.rs:242-243`), making any status derived from
that field a constant.

### Public and private transactions expose different things

Private state exists on-chain only as commitments, so plaintext travels inside
the transaction, whereas public state lives in the replicated state machine and
can be queried. Private-transaction post-states are carried in
`encrypted_post_state` and are readable only by the viewing key holder
(`lez/indexer/service/protocol/src/lib.rs:220-253`).

The consequence for the integrator profile is a hard boundary that the API must
state rather than paper over: deposits into private accounts are not trackable
from indexer data by a third party. The public leg of a shielded transaction is
readable, so a deshield into a public account is trackable. An exchange
integrating LEZ credits public accounts.

### Programs emit nothing

`ProgramOutput` carries state and chained calls and has no event or log field,
so there is nothing for an event query to return. This RFP does not introduce an
event system, which is a runtime change rather than an API one; the effects API
is the substitute, because a persisted state diff describes what a transaction
did without the program having to declare it.

### Retention is reported, not configured

The requirements below ask for the retention floor to be reported rather than
for pruning to be added: EIP-4444's design lesson is that retention should be
declared rather than discovered through a failed request
([Appendix: Blockchain API and SDK Ecosystem, section 1.29](../appendix/blockchain-api-sdk-ecosystem.md#129-read-historical-state-at-a-past-version)).

## ✅ Scope of Work

### Hard Requirements

#### Functionality

Each requirement carries a readiness marker describing its starting point in the
current LEZ codebase. **Ready** means the capability is already exported by the
indexer FFI and only a specification, test, or documentation obligation remains.
**Ready, not exposed** means it exists in the indexer service, RPC, or store but
the FFI does not export it, so the work is exposure. **Computed, not persisted**
means the indexer derives the data during ingestion but neither stores nor
exposes it, so the work is persistence plus exposure. **New** means neither the
data nor the capability exists today. The markers indicate effort, not priority:
every hard requirement is required regardless of its marker.

01. The FFI exposes a per-transaction effects query that, given a transaction
    hash, returns the state diff that transaction produced: the accounts whose
    public state changed, their pre-state and post-state balances and nonces,
    the new commitments, the new nullifiers, and the program invoked. The diff
    is the one computed at ingest by `execute_on_state`
    (`lez/chain_state/src/apply.rs:159`), persisted rather than recomputed on
    read. **[Computed, not persisted]**
02. The effects query returns a balance delta per affected account, expressed as
    pre-state and post-state values, for public transactions and for the public
    leg of privacy-preserving transactions. **[Computed, not persisted]**
03. The effects query reports, for a privacy-preserving transaction, which parts
    of its state change are not readable without the viewing key, distinctly
    from reporting that no state changed. **[New]**
04. The FFI exposes a simulation query that takes an unsubmitted transaction,
    executes it against the indexer's current finalised state without applying
    it, and returns the same diff type the effects query returns, plus an
    execution outcome: success, or typed failure. **[Computed, not persisted]**
05. The simulation query returns the block identifier its result was computed
    against, so a caller can tell what the answer is an answer about. **[New]**
06. The FFI exposes an account read pinned to a block identifier, returning the
    account record as it stood at that block. This exports the existing
    `getAccountAtBlock` indexer RPC method
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[Ready, not exposed]**
07. The FFI exposes a batch account read taking a list of account identifiers
    and returning a result per identifier in request order, backed by
    `multi_get_cf` as the block and transaction reads in the same store already
    are. **[New]**
08. The batch account read accepts an optional block identifier that pins every
    account in the batch to the same block, so a multi-account read is
    internally consistent under concurrent ingestion. **[New]**
09. Every account read, single or batch, current or pinned, distinguishes an
    account that has never been seen from an account holding zero balance and
    zero nonce. The current `query_account` cannot: the store returns a default
    record for an unknown identifier
    ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
    **[New]**
10. The FFI exposes a per-transaction status query returning a level from a
    documented set that distinguishes at minimum: not known to the indexer,
    present in an indexed block, and final. The level is derived from the
    transaction's presence in the store and its block's position relative to the
    indexed tip, and not from `Block.bedrock_status`, which the indexer
    overwrites to `Finalized` unconditionally
    (`lez/indexer/core/src/block_store.rs:242-243`). **[New]**
11. The per-transaction status query returns the identifier of the block
    containing the transaction, and the indexed tip the level was assessed
    against. **[Ready, not exposed]**
12. The per-transaction status query reports execution success or failure, with
    a typed reason on failure, for any transaction the indexer has executed.
    **[Computed, not persisted]**
13. The FFI exposes a subscription to finalised blocks that delivers each newly
    indexed block identifier to a registered consumer, exporting the existing
    `subscribeToFinalizedBlocks` indexer RPC method. The consumer is notified
    through a callback rather than by polling. **[Ready, not exposed]**
14. The block subscription accepts a start position: a block identifier the
    consumer last processed. Delivery resumes from the block after that
    position, so a consumer that reconnects observes no gap. The current RPC
    subscription takes no arguments
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.33](../appendix/blockchain-api-sdk-ecosystem.md#133-resume-a-stream-from-a-known-position)).
    **[New]**
15. The FFI exposes an account transaction count, so a caller paginating
    `query_transactions_by_account` can tell how many entries exist and whether
    it has reached the end. **[Ready, not exposed]**
16. `query_transactions_by_account` accepts an ordering parameter supporting
    both oldest-first and newest-first. Newest-first is the order a deposit
    tracker reads in. **[New]**
17. The `before` parameter on `query_block_vec` is documented as exclusive, and
    every paginated response reports if more remain. **[Ready, not exposed]**
18. The FFI exposes the chain tip as a single call returning the block
    identifier together with the block's height, timestamp, and hash, so
    learning about the tip does not cost a second call. **[Ready, not exposed]**
19. The FFI exposes the zone identifier the indexer is reading, so an
    application can confirm which zone it is connected to. It is derived from
    indexer data, not the sequencer's `getChannelId`. **[Ready, not exposed]**
20. The FFI exposes the indexer's retention floor: the earliest block for which
    account state can be read at a pinned block identifier, and the earliest
    block for which a transaction can be retrieved. The current implementation
    never prunes, so this may be genesis; the requirement is that the value is
    reported rather than assumed. **[New]**
21. Every function added by this RFP is exposed through `lez_indexer_module`
    with the same semantics, including the not-found and error distinction from
    Functionality #22. No capability added to the FFI stops at the module
    boundary. **[New]**
22. The FFI and the module signal not-found, invalid-argument, and backend
    failure as three distinguishable outcomes on every query. The module
    currently flattens not-found and failure into an empty string
    ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
    **[New]**
23. Errors carry an application code from a documented, stable code space, a
    category, and a retryability signal. Two failure causes that require
    different caller recovery do not share a code. The current implementation
    uses the stock JSON-RPC `InternalError` code with free text
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).
    **[New]**
24. The FFI exposes a machine-readable description of its own surface, covering
    every exported function, its parameters, its return type, and its error
    codes. The existing `getSchema` describes the block type only and is not
    exported
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**

#### Usability

1. Provide a Logos module surface (`lez_indexer_module`) covering every function
   in Functionality, usable to build Logos modules for reading LEZ state,
   following the chain, and tracking deposits.
2. Provide a CLI that covers core functionality: read an account at current and
   at a pinned block, read a batch of accounts, read a block, read a
   transaction, read a transaction's effects, simulate a transaction, query a
   transaction's status, list an account's transactions with ordering and
   pagination, and follow the chain tip. The CLI may have fewer features than
   the FFI but must support all essential operations.
3. Provide a worked deposit-tracking example, in the CLI or as a documented
   reference consumer, that follows the chain from a chosen start block, detects
   credits to a supplied set of public accounts, and reports each credit with
   its transaction hash, amount, block, and status level.
4. Document, per function, which capability of the surveyed ecosystem it
   corresponds to, referencing the numbered functions in
   [Appendix: Blockchain API and SDK Ecosystem, section 1](../appendix/blockchain-api-sdk-ecosystem.md#1-api-functions).
5. Document which state is readable and which is not for privacy-preserving
   transactions, and state that deposits into private accounts are not trackable
   from indexer data without the viewing key.
6. Return clear, actionable error messages for every failure mode, each mapped
   to the code space required by Functionality #23.
7. Document the semantics of every pagination parameter, including the
   exclusivity of `before`, the stability of both cursors, and the behaviour
   when new data lands ahead of an offset.

#### Reliability

1. A pinned account read at a given block identifier returns the same value on
   every call, however far ingestion has advanced. **[Ready, not exposed]**
2. A batch account read pinned to a block identifier returns a set of accounts
   consistent with one another as at that block, not a mixture of values read at
   different points during concurrent ingestion. **[New]**
3. Pagination cursors remain valid across ingestion: a caller walking blocks by
   cursor or an account's transactions by offset neither skips nor repeats an
   entry that existed when the walk began.
4. A subscription consumer that disconnects and reconnects with its last
   processed block identifier receives every block after that position, with no
   gap and no assumption that the consumer was connected.
5. The simulation query never mutates indexer state, and a simulation failure
   leaves the store unchanged.
6. Every read reports the indexed tip it was served against, so a caller can
   detect that it read from a stalled or lagging indexer.
7. A stalled indexer is reported as stalled by the status surface rather than
   serving stale reads silently.

#### Performance

1. Document the latency of a pinned account read as a function of distance from
   the nearest state snapshot. `getAccountAtBlock` replays up to 99 blocks from
   the nearest breakpoint, each running the RISC0 executor, so the worst case
   must be measured rather than estimated.
2. A batch account read of N accounts costs materially less than N single reads,
   and the improvement is measured and documented for representative N.
3. Document the storage cost of persisting per-transaction effects: bytes per
   transaction, and projected growth against a stated block rate and transaction
   density.
4. Document the latency of the simulation query for a representative
   transaction, separated into executor time and state access time.
5. Benchmarks are reproducible from the test suite and run against a LEZ
   devnet/testnet indexer.

#### Supportability

1. The FFI and module are built and tested against a LEZ devnet/testnet indexer.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   with an indexer ingesting from it, and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
5. Tests cover, at minimum: a never-seen account distinguished from a zero
   account; a pinned read at a block before and after a state change; a batch
   read spanning present and absent accounts; effects for a public transaction,
   for the public leg of a privacy-preserving transaction, and for a
   privacy-preserving transaction whose private leg is unreadable; a simulation
   that succeeds and one that fails; a transaction status at each documented
   level; and a subscription resumed from a stored position across a
   disconnection.
6. A README documents end-to-end usage: building the FFI and module, pointing
   them at an indexer, and step-by-step instructions for every operation via the
   CLI.
7. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the FFI and module, covering the developer integration journey for
   reading state, tracking deposits, and confirming transactions.
8. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator journey.
9. Provide a compatibility note stating which functions are additive to the
   existing eight queries and which change the behaviour of an existing one, so
   existing consumers of `lez_indexer_module` can assess the upgrade.

#### + Privacy

1. No function added by this RFP returns plaintext private state, a viewing key,
   or any value derived from either. Private post-states remain readable only by
   the viewing key holder.
2. The effects and status surfaces do not report, for a privacy-preserving
   transaction, any linkage between its private inputs and its private outputs
   beyond the commitments and nullifiers already published on-chain.
3. Provide a privacy properties document covering: what a third party reading
   the indexer can learn about a public transaction, about the public leg of a
   privacy-preserving transaction, and about its private leg; which of those
   boundaries are enforced by encryption and which by the API declining to
   expose data it holds; and what an operator running their own indexer can see
   that a remote caller cannot.

### Soft Requirements

If possible.

#### Functionality

1. A subscription filtered to a supplied set of accounts, delivering only blocks
   containing a transaction that touches one of them. Every surveyed chain that
   offers a block subscription also offers a filtered one
   ([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).
2. A batch transaction read, taking a list of transaction hashes and returning a
   result per hash in request order.
3. A block-scoped effects query returning the diffs for every transaction in one
   block, so a consumer following the chain reads one result per block rather
   than one per transaction.
4. A decoder for the account data blob returned by the account read, so a caller
   can interpret token balances without a client-side Borsh decode against an
   unpublished schema. `getAccount` returns program data as an opaque base64
   blob of up to 100 KiB, with the token balance inside `Account.data` rather
   than `Account.balance`.
5. A retention configuration for the indexer, so an operator can bound storage
   growth. The retention floor reported per Functionality #20 then becomes a
   moving value rather than genesis.

### Out of Scope

The following are explicitly excluded from this RFP:

- **Wallet and key management.** Key handling, derivation, watch-only address
  derivation, viewing keys, and signing belong to the LEZ wallet FFI, the second
  of the six deliverables.
- **Transaction construction and submission.** Building, signing, and submitting
  transactions run through the wallet path (`wallet_ffi` and `lez_core`). The
  indexer stack performs no writes and this RFP does not change that.
- **The JSON-RPC proxy and the language SDKs.**
  [logos-co/ecosystem#220](https://github.com/logos-co/ecosystem/issues/220)
  covers the transport, the wallet SDK, and the indexer SDK. Further transport
  bindings such as gRPC and GraphQL are
  [logos-co/ecosystem#222](https://github.com/logos-co/ecosystem/issues/222).
  This RFP defines what those consume, not how it is transported or wrapped.
- **Bridging sequencer methods.** `getAccountBalance`, `getAccountsNonces`,
  `getProofsAndRoot`, `getProgramIds`, and `getChannelId` are sequencer methods.
  Where an equivalent capability is required above, it is required to be derived
  from indexer data. The sequencer is treated as a disappearing black box, per
  its own source comment (`lez/sequencer/service/rpc/src/lib.rs:46-47`).
- **A mempool or pending-transaction view.** The sequencer's mempool is an
  in-process `tokio::sync::mpsc` channel with no query surface, and no path to
  it exists anywhere in the indexer stack. Pending visibility is net-new work
  that depends on the intra-sequencer mempool landing first, and belongs in a
  later RFP.
- **A program event or log system.** `ProgramOutput` carries no event field and
  never reaches the indexer. Adding events is a runtime change, not an API one.
  The effects API in Functionality #1 is the substitute for the reads events
  would have served.
- **Fee and cost estimation.** LEZ transactions carry no fee field and execution
  is bounded by a fixed cycle cap, so there is no cost to estimate
  ([Appendix: Blockchain API and SDK Ecosystem, section 1.16](../appendix/blockchain-api-sdk-ecosystem.md#116-estimate-execution-cost)).
- **Tracking deposits into private accounts.** Private post-states are encrypted
  and readable only by the viewing key holder, so this is foreclosed by the
  design of the chain rather than by the scope of this RFP.

## ⚠ Platform Dependencies

This RFP has no hard blockers. Every capability required above is either already
present on the indexer RPC and unexported, already computed by the indexer and
discarded, or derivable from data the indexer store already holds. The
frontmatter `dependencies` list is therefore empty.

### Soft blockers

#### Intra-sequencer mempool

Pending-transaction visibility, excluded above, becomes possible only once the
sequencer's mempool has an external query surface. Nothing in this RFP depends
on it, and a proposal must not assume it.

### Risks

#### Pinned-read latency

`getAccountAtBlock` replays up to 99 blocks from the nearest state snapshot,
each running the RISC0 executor. The worst-case latency is not determinable from
source and may be high enough that a pinned read is unsuitable for a hot path.
Performance requirement #1 requires it to be measured; if the measurement is
poor, the mitigation is a denser snapshot interval or a per-account index, and a
proposal should state which it would pursue.

#### Snapshot semantics under concurrent ingestion

RocksDB `multi_get_cf` snapshot behaviour under concurrent ingestion is not
settled by reading the source alone. A torn paging read is unlikely but not
excluded, and Reliability requirements #2 and #3 are the conditions a proposal
must demonstrate it meets, by explicit snapshotting if necessary.

#### Storage growth

The indexer never prunes and writes a full state snapshot every 100 blocks.
Persisting per-transaction effects adds to that. Performance requirement #3
requires the added cost to be quantified so an operator can size for it, and
soft Functionality #5 offers retention configuration as the mitigation.

## 👤 Recommended Team Profile

Team experienced with:

- Rust systems development, particularly FFI boundaries and C ABI design
- Blockchain node and indexer internals
- RocksDB or comparable embedded key-value stores, including snapshot and
  consistency semantics
- API design and versioning for third-party integrators
- C++ interoperability, for the module layer above the FFI
- Exchange or custodian integration work, or comparable experience building
  against a chain's read API in production

## ⏱ Timeline Expectations

Estimated duration: **10–14 weeks**.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md): as-built
  inventory of the LEZ indexer FFI and RPC, the sequencer RPC, the wallet FFI
  and `lez_core` module, and the L1 bindings, routes, and module
- [Appendix: Blockchain API and SDK Ecosystem](../appendix/blockchain-api-sdk-ecosystem.md):
  34 API functions across eight established chains, with transports, SDK
  languages, response shapes, and per-function gap notes for LEZ
- [logos-execution-zone](https://github.com/logos-blockchain/logos-execution-zone):
  the LEZ sequencer, indexer, and wallet
- [lez-indexer-module](https://github.com/logos-blockchain/lez-indexer-module):
  the Logos Core module wrapping the indexer FFI
- [Logos Documentation](https://github.com/logos-co/logos-docs)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
