---
id: RFP-027
title: LEZ Indexer FFI
tier: L
status: open
category: Developer Tooling & Infrastructure
dependencies: []
---

# RFP-027 — LEZ Indexer FFI

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

### Target architecture and SDK suite

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
   API, in the sense that the `eth` namespace is on Ethereum's JSON-RPC. The indexer
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

### The indexer is the only consumer API

The indexer is the whole of the read surface available to a consumer. The
sequencer is an internal component of LEZ, not an API target, and neither this
RFP nor the others in this set address its interface.

The indexer serves every read, acting as a cache and a proxy for on-chain data.
Where it does not already hold what a consumer asks for, it obtains it from
wherever inside LEZ that data lives, the sequencer included, rather than
directing the consumer there. Which component answers a read internally is not a
property the API exposes, and it is free to change: indexer to sequencer
communication is expected to move from JSON-RPC to a libp2p mempool, and no
consumer should be able to tell.

A capability a consumer needs is therefore required of the indexer regardless of
which component holds the data today. The pending set is the clearest case, and
the same reasoning governs every read below.

## 🔥 Why This Matters

LEZ is adopted when existing projects integrate it. To increase the value a
project sees in integrating LEZ, we need to decrease the upfront cost by
providing the software they need.

So the requirements below are set by what those integrators already rely on
elsewhere. Nine established chains were surveyed for that purpose. LEZ does not
currently meet several of the norms: there are no per-transaction effects; no
confirmation level; and no subscription reaches the FFI, which is the boundary
an integrator can actually call. Simulation is the one norm this RFP declines
rather than fills, for the reason given in the Design Rationale. See
[Appendix: Blockchain API and SDK Ecosystem, section 4](../appendix/blockchain-api-sdk-ecosystem.md#4-gap-summary).

Most of the missing capability already exists one layer below the FFI, as the
readiness markers below record, so the work is largely exposure rather than
invention.

## 🏗 Design Rationale

### The target is LEZ after testnet 0.3

This RFP specifies the surface for LEZ as it will be after testnet 0.3, not as
it is on the default branch today. Testnet 0.3 brings gas, so transactions carry
a fee and execution has a price a caller pays. The requirements below assume
that: cost is something to estimate, report, and budget against, and the
readiness markers describe the starting point for the work rather than a
constraint on what the surface may express.

### Simulation belongs to the wallet, not to the indexer

Simulation is the one capability an integrator expects from a node API that this
RFP does not require. Six of the nine surveyed chains execute an unsubmitted
transaction and return its outcome, and an RPC provider elsewhere is expected to
offer it
([Appendix: Blockchain API and SDK Ecosystem, section 1.15](../appendix/blockchain-api-sdk-ecosystem.md#115-simulate-transaction-execution)).
On LEZ that shape does not fit.

A privacy-preserving transaction is executed by the wallet, which runs the
program locally over notes only it can decrypt and submits a proof that the
execution was correct. The proof is an input to the transaction rather than a
result of executing it, so a node holds neither the witness material nor the
plaintext a simulation would need, and there is nothing for the indexer to
simulate before the wallet has already done the work. Zcash reaches the same
conclusion from the same premise, and exposes no simulation on either leg.

Public execution could be simulated by the indexer, but siting it there would
split one capability across two components and answer against finalised state
rather than the state the caller is building on. The wallet already holds the
executor for the private path and can read whatever state it needs through the
queries below, so it can answer for both kinds of transaction, against a state
it chose, at the moment it is constructing the transaction. That is also what
simulation is for elsewhere: Stellar returns the transaction data and the
minimum resource fee that the caller copies back into the transaction before
submitting, which makes simulation a construction step rather than a read.
Construction is out of scope here and belongs to the wallet FFI, the second of
the six deliverables.

### A block identifier is a height, and a height is not a name

`BlockId` is a `u64` counting from a genesis of 1
(`lee/state_machine/core/src/lib.rs:36-38`), and the codebase treats it
arithmetically throughout: pagination descends by subtracting one
(`lez/storage/src/indexer/read_multiple.rs:11`), state snapshots are indexed by
dividing it by the breakpoint interval (`lez/storage/src/indexer/mod.rs:293-298`),
and the sequencer assigns a local named `new_block_height` straight into the
field (`lez/sequencer/core/src/lib.rs:1198`). The block hash is the separate
32-byte value the header also carries (`lez/common/src/block.rs:51-58`). Storage
reflects the difference: the height is the primary key blocks are stored under,
and the hash is a secondary index onto it
(`lez/storage/src/indexer/read_once.rs:43-46`, `read_once.rs:68-71`).

The distinction matters because a height names a position rather than a block.
The sequencer's own reorg handling describes inscribing a second block at a
height the channel already holds (`lez/sequencer/core/src/lib.rs:595-873`), so
one identifier can resolve to different blocks at different times, while a hash
resolves to one block or to none. The API therefore uses the two for different
purposes: a height to say where in the chain a caller is reading, and a hash to
say which block a caller read. Where a caller needs to know that the answer it
holds still refers to the block it was given, both travel together.

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
(`lez/indexer/service/protocol/src/lib.rs:193-260`).

The consequence for the integrator profile is a hard boundary that the API must
state rather than paper over: deposits into private accounts are not trackable
from indexer data by a third party. The public leg of a shielded transaction is
readable, so a deshield into a public account is trackable. An exchange
integrating LEZ credits public accounts.

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
every hard requirement is required regardless of its marker. When not already
implemented, the function signatures below should be seen as suggestions; the
description of functionality underneath is the requirement.

Every exported function takes the indexer handle returned by `start_indexer` as
its first argument, and a null handle is signalled as
`OperationStatus::NullPointer` rather than as a not-found result. The signatures
below elide it.

##### Transaction effects

The read that says what a transaction did to balances
([Appendix: Blockchain API and SDK Ecosystem, section 1.27](../appendix/blockchain-api-sdk-ecosystem.md#127-get-execution-effects-and-state-changes)).

- **Ethereum**: `eth_getTransactionReceipt`, with effects inferred from logs.
- **Solana**: `getTransaction`, returning pre-balances and post-balances
  directly.
- **Bitcoin**: no equivalent; the effect is the UTXO set delta itself.
- **Zcash**: `z_viewtransaction` for the shielded leg, `getblockdeltas` and
  `getspentinfo` for the transparent one, the latter two being insight-explorer
  methods a node must be started and reindexed for. The shielded call is
  wallet-layer, documented as returning "detailed shielded information about
  in-wallet transaction", so it reads only what the node holds keys for. A third
  party sees transparent effects openly and shielded effects not at all, and no
  parameterisation of the call changes that, the entitlement being cryptographic
  rather than an access-control setting.

**`query_transaction(hash) -> PointerResult<FfiOption<FfiTransaction>, OperationStatus>`**

Returns the transaction carrying the given hash, as stored in the block that
holds it. Absence is signalled structurally, through `FfiOption`, and not
through the status channel.

1. The query returns the stored transaction for a known hash and an absent
   `FfiOption` for a hash no indexed block holds, keeping the two distinct from
   a backend failure. **[Ready]**
2. The returned transaction discriminates public, privacy-preserving, and
   program-deployment bodies, exposing for each the fields the wire type carries
   (`lez/indexer/service/protocol/src/lib.rs:193-260`). **[Ready]**

**`query_transaction_effects(hash) -> PointerResult<FfiOption<FfiStateDiff>, OperationStatus>`**

Returns the state diff a transaction produced, read from storage rather than
recomputed on the caller's behalf.

3. The FFI exposes a per-transaction effects query that, given a transaction
   hash, returns the state diff that transaction produced: the accounts whose
   public state changed, their pre-state and post-state balances and nonces, the
   new commitments, the new nullifiers, and the program invoked. The diff is the
   one computed at ingest by `execute_on_state`
   (`lez/chain_state/src/apply.rs:159`), persisted rather than recomputed on
   read. **[Computed, not persisted]**
4. The effects query returns a balance delta per affected account, expressed as
   pre-state and post-state values, for public transactions and for the public
   leg of privacy-preserving transactions. **[Computed, not persisted]**

**`query_events(from_block, to_block, tx_hash, program_id, selector) -> PointerResult<FfiVec<FfiEventRecord>, OperationStatus>`**

Returns the program events matching the filter. A non-null `tx_hash` makes the
call a point lookup and the block range is ignored; otherwise the range runs
from `from_block` to `to_block`, defaulting to the indexed tip.

5. The query returns each matching event record with its block identifier,
   transaction index, transaction hash, emitting program, selector, and data
   payload. **[Ready]**
6. A range query spanning more than `MAX_EVENT_QUERY_BLOCK_SPAN` blocks, a bound
   past the indexed tip, an inverted range, and a range outside the indexer's
   event-filter history each return `InvalidArgument` rather than an empty
   result. **[Ready]**
7. Events are absent for privacy-preserving transactions, `ProgramOutput.events`
   being dropped for function privacy, so the query answers for the public leg
   only. **[Ready]**

##### Account reads

Reading account state, at the tip and at a past block, singly and in batches
([Appendix: Blockchain API and SDK Ecosystem, section 1.7](../appendix/blockchain-api-sdk-ecosystem.md#17-get-an-account-or-object)).

- **Solana**: `getAccountInfo` and `getMultipleAccounts`, the closest shape to
  what is required here.
- **Ethereum**: split across `eth_getCode` and `eth_getStorageAt`, with no
  single account object.
- **Bitcoin**: `gettxout` reads a UTXO; the model has no account record.
- **Zcash**: `getaddressbalance` for transparent addresses, readable by any
  party, subject to the node running with the insight-explorer options;
  `z_getbalanceforviewingkey` for shielded ones, documented as returning the
  balance "viewable by a full viewing key known to the node's wallet". Two
  conditions sit on the shielded read: the caller holds the key, and the key was
  already imported into that node's wallet. There is no shielded read a party
  without a key can make.

**`query_account(account_id) -> PointerResult<FfiAccount, OperationStatus>`**

Returns the account record as it stands at the indexer's current state: its
owning program, balance, nonce, and program data blob.

8. The query returns the account's `program_owner`, `balance`, `nonce`, and
   `data` blob, balance and nonce carried as little-endian 16-byte values.
   **[Ready]**
9. An identifier the state holds no record for reads as the default account,
   every field at its zero value, which is also what an uninitialised account
   holds. The API documents that the read answers what the account holds
   rather than whether it exists, and that a default `program_owner` marks an
   account as unclaimed, the same test the state machine applies before
   allowing one to be modified
   (`lee/state_machine/src/validated_state_diff/mod.rs:351-365`). **[Ready]**
10. The account reads answer for public state alone, and the API says so. A
    private balance has no account record to return: private state is a
    commitment set and a nullifier set carrying no account identifier
    (`lee/state_machine/src/state/mod.rs:115-116`), so a default record means
    the state holds no public balance for that identifier and is not evidence
    that the party holds nothing. **[New]**

**`query_account_at_block(account_id, block_id) -> PointerResult<FfiAccount, OperationStatus>`**

Returns the account record as it stood at a given block, rather than at the tip.

11. The FFI exposes an account read pinned to a block identifier, returning the
    account record as it stood at that block, on the same terms as the read at
    the tip. This exports the existing `getAccountAtBlock` indexer RPC method
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[Ready, not exposed]**
12. The pinned read returns the hash of the block it answered against, and the
    API documents that the read pins to a height rather than to a block: the
    same identifier can resolve to a different block after a reorg, and the hash
    is what tells a caller which one it read. **[New]**

**`query_accounts(account_ids, block_id) -> PointerResult<FfiVec<FfiAccount>, OperationStatus>`**

Returns one result per requested account identifier, optionally pinned to a
block.

13. The FFI exposes a batch account read taking a list of account identifiers
    and returning a result per identifier in request order, backed by
    `multi_get_cf` as the block and transaction reads in the same store already
    are. An identifier the state holds no record for reads as the default
    account rather than failing the call. **[New]**
14. The batch account read accepts an optional block identifier that pins every
    account in the batch to the same block, so a multi-account read is
    internally consistent under concurrent ingestion. **[New]**

##### Commitment membership proofs

Reading the inclusion proofs a caller needs to spend a private note, and the
root they are proven against. This read has no counterpart among the surveyed
chains, the commitment set being a structure only a shielded chain carries.

**`query_commitment_proofs(commitments) -> PointerResult<FfiCommitmentProofs, OperationStatus>`**

Returns a membership proof per requested commitment, together with the
commitment set root they are proven against.

15. The FFI returns, for a list of commitments, a membership proof per
    commitment in request order and the commitment set root those proofs are
    proven against, both read from indexer data. A commitment the set does not
    hold is reported as absent per entry rather than failing the call. This is
    the data the sequencer's `getProofsAndRoot` returns today
    (`lez/sequencer/service/rpc/src/lib.rs:80-84`), which the wallet reaches the
    sequencer for (`lez/wallet/src/lib.rs:660-667`). A consumer that holds a
    viewing key needs it to spend a note, and under the rationale above it is
    required of the indexer regardless of which component holds it. **[New]**
16. The proofs and the root returned by one call are consistent with one
    another: every proof verifies against the returned root. **[New]**

##### Transaction status

Answering how far a transaction has progressed and whether it succeeded
([Appendix: Blockchain API and SDK Ecosystem, section 1.24](../appendix/blockchain-api-sdk-ecosystem.md#124-get-transaction-status)).

- **Ethereum**: `eth_getTransactionReceipt`, whose absence means not yet mined.
- **Solana**: `getSignatureStatuses`, carrying an explicit commitment level.
- **Bitcoin**: `gettransaction` reports confirmations but is wallet scoped.
- **Zcash**: `gettransaction` for confirmations, wallet scoped as on Bitcoin.
  ZIP 315 specifies the confirmation policy, three confirmations for trusted
  funds and ten for untrusted, as wallet behaviour rather than as a node call.

**`query_transaction_status(hash) -> PointerResult<FfiTransactionStatus, OperationStatus>`**

Returns how far a transaction has progressed, where it sits, and whether its
execution succeeded.

17. The FFI exposes a per-transaction status query returning a level from a
    documented set that distinguishes at minimum: not known to the indexer,
    present in an indexed block, and final. The level is derived from the
    transaction's presence in the store and its block's position relative to the
    indexed tip, and not from `Block.bedrock_status`, which the indexer
    overwrites to `Finalized` unconditionally
    (`lez/indexer/core/src/block_store.rs:242-243`). **[New]**
18. The per-transaction status query returns the identifier of the block
    containing the transaction, and the indexed tip the level was assessed
    against. **[Ready, not exposed]**
19. The per-transaction status query reports execution success or failure, with
    a typed reason on failure, for any transaction the indexer has executed.
    **[Computed, not persisted]**

**`query_status() -> *mut c_char`**

Returns the indexer's own ingestion state as a JSON document: `state`,
`indexed_block_id`, `last_error`, `stall_reason`, `cross_zone_halt`, and
`cross_zone_peers`.

20. The query reports the ingestion state as one of `Starting`, `Syncing`,
    `CaughtUp`, `Error`, `Stalled`, or `Halted`, together with the indexed block
    identifier, and carries the stall and cross-zone diagnostics when ingestion
    stopped. A state a client build does not recognise is treated as not known
    healthy rather than as a decode failure. **[Ready]**
21. The status query signals a null handle and a serialisation failure as
    distinguishable outcomes rather than returning a bare null pointer for both
    ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
    **[New]**

##### Pending transactions

Listing what the network holds but has not yet included in a block
([Appendix: Blockchain API and SDK Ecosystem, section 1.26](../appendix/blockchain-api-sdk-ecosystem.md#126-inspect-the-mempool-or-pending-set)).

- **Bitcoin**: `getrawmempool` lists the set, `getmempoolentry` returns one
  entry's detail, `getmempoolinfo` its size.
- **Cosmos**: `/unconfirmed_txs` and `/num_unconfirmed_txs`.
- **Ethereum**: `eth_newPendingTransactionFilter`, with no direct dump.
- **Solana, XRPL, Stellar, NEAR, Sui**: no equivalent.

A privacy-preserving transaction commits to a post-state when it is built, so a
pending transaction that changes the pre-state it assumed invalidates it before
it is ever submitted. Generating the proof takes minutes, so a wallet that can
see the pending set aborts a proof it now knows will be rejected instead of
finishing it. That is the difference between reading the pending set and
diagnosing a failure afterwards.

**`query_pending_digests(cursor, limit) -> PointerResult<FfiVec<FfiPendingDigest>, OperationStatus>`**

Lists the pending set as digests: per entry, its identifier and the account
identifiers, nonces, and nullifiers it touches.

22. The FFI lists pending transactions as digests carrying, per entry, its
    identifier together with the account identifiers, nonces, and nullifiers the
    transaction acts on, so a caller detects a conflict with a transaction it is
    about to build without fetching bodies. **[New]**
23. The listing is bounded and paginated by cursor, on the same terms as the
    other paginated reads, and reports whether more entries remain. The pending
    set turns over as entries are included or dropped, so an offset into it
    names a different entry from one call to the next. **[New]**
24. The digest listing is cheap enough to poll for the duration of a proof
    generation, so a wallet re-checks for a conflict while proving and abandons
    the proof rather than completing one it knows will be rejected. **[New]**

**`query_pending_entry(id) -> PointerResult<FfiOption<FfiTransaction>, OperationStatus>`**

Returns one pending transaction in full, for an identifier taken from the digest
listing.

25. The FFI returns a pending transaction's body for an identifier from the
    digest listing, in the same shape `query_transaction` returns an included
    one, and reports absence distinctly from failure: an entry included or
    dropped between the two calls is gone rather than an error. **[New]**
##### Subscriptions

Pushing new blocks to a consumer instead of making it poll
([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).

- **Ethereum**: `eth_subscribe("newHeads")`.
- **Solana**: `slotSubscribe` and `blockSubscribe`.
- **Bitcoin**: the ZeroMQ publishers `-zmqpubhashblock` and `-zmqpubrawblock`.
- **Zcash**: no JSON-RPC subscription; the node's own push surface is ZeroMQ,
  without the `-zmqpubsequence` publisher Bitcoin offers for loss detection. The
  streaming interface an integrator uses sits on lightwalletd rather than the
  node: `GetBlockRange` streams compact blocks over gRPC, taking an explicit
  height range, so a consumer resumes by asking for the range it has not yet
  seen.

**`subscribe_to_finalized_blocks(from_block, callback, user_data) -> PointerResult<FfiSubscription, OperationStatus>`**

Registers a consumer that is called with each newly indexed block, optionally
resuming from a position the consumer already processed.

26. The FFI exposes a subscription to finalised blocks that delivers each newly
    indexed block to a registered consumer, exporting the existing
    `subscribeToFinalizedBlocks` indexer RPC method. The consumer is notified
    through a callback rather than by polling. **[Ready, not exposed]**
27. Each delivery carries the block's height and its hash together, so a
    consumer knows which block occupied the position it was told about without a
    second read. The existing subscription yields a height alone
    (`lez/indexer/service/rpc/src/lib.rs:44`), which does not identify a block
    across a reorg; the pair already exists as `BlockMeta`
    (`lez/common/src/block.rs:11-14`). **[New]**
28. The block subscription accepts a start position: a block identifier the
    consumer last processed. Delivery resumes from the block after that
    position, so a consumer that reconnects observes no gap. Neither existing
    RPC subscription carries one: `subscribeToFinalizedBlocks` takes no
    arguments, and `subscribeToEvents` filters by transaction, program, and
    selector but not by position
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.33](../appendix/blockchain-api-sdk-ecosystem.md#133-resume-a-stream-from-a-known-position)).
    **[New]**

**`unsubscribe(subscription) -> OperationStatus`**

Cancels a registered subscription and releases the resources it holds.

29. Cancelling a subscription stops delivery, and no callback fires after the
    call returns. **[New]**

##### Pagination and account history

Walking an account's transaction history in bounded pages
([Appendix: Blockchain API and SDK Ecosystem, section 1.30](../appendix/blockchain-api-sdk-ecosystem.md#130-paginate-a-result-set)).

- **Solana**: `getSignaturesForAddress`, taking `before`, `until`, and `limit`.
- **Bitcoin**: `listtransactions`, using count and skip.
- **Ethereum**: no cursor scheme on the standard node API; `eth_getLogs` is
  bounded by block range instead.
- **Zcash**: `getaddresstxids` for transparent addresses, insight-explorer gated
  and bounded by an inclusive start and end height rather than a cursor;
  `z_listreceivedbyaddress` for shielded ones, reachable only for keys the node
  holds and taking an `asOfHeight` parameter that pins the read to a past
  height. Height bounding rather than cursoring leaves a caller re-deriving its
  position from heights it has already scanned, and no method returns a next
  position.

**`query_transactions_by_account(account_id, cursor, limit, order) -> PointerResult<FfiVec<FfiTransaction>, OperationStatus>`**

Returns one page of the transactions touching an account, resumed from a cursor
the previous page returned. Exported today as a numeric offset into the
per-account index.

30. The query returns at most `limit` transactions from the position the cursor
    names, or from the start of the walk when the cursor is absent, and stops
    early at the end of the account's history rather than failing. The bounded
    read exists; the cursor replaces the offset it takes today. **[New]**
31. `query_transactions_by_account` accepts an ordering parameter supporting
    both oldest-first and newest-first. Newest-first is the order a deposit
    tracker reads in. **[New]**
32. The page is walked by an opaque cursor rather than by a numeric offset, and
    the cursor remains stable across ingestion: a caller resuming from one
    neither skips nor repeats an entry that existed when the walk began, however
    many transactions have landed since. An offset into a growing index cannot
    hold that property, because entries arriving ahead of the offset shift every
    later position, and a deposit tracker reading newest-first is the case that
    breaks first. The cursor is opaque to the caller, which may not construct
    one or infer a position from it. **[New]**

**`query_blocks(from, limit, order) -> PointerResult<FfiVec<FfiBlock>, OperationStatus>`**

Returns a page of blocks starting at `from`, walked in the requested direction,
or starting at the indexed tip when `from` is absent. Exported today as
`query_block_vec`, which descends only.

33. The query returns at most `limit` blocks, walked from `from`, or from the
    indexed tip when `from` is absent. **[Ready]**
34. The bound is documented as exclusive. The store already implements it
    that way when descending, from `before_id.saturating_sub(1)`
    (`lez/storage/src/indexer/read_multiple.rs:11`), leaving the documentation
    obligation only. **[Ready]**
35. `query_blocks` accepts an ordering parameter supporting both oldest-first
    and newest-first, on the same terms as
    `query_transactions_by_account`. Oldest-first is the order a consumer
    scanning forward reads in: a wallet syncing private accounts walks
    ascending from the last block it processed to the tip, decrypting each
    privacy-preserving transaction body against its own viewing key. Descending
    from the tip cannot serve that walk. **[New]**
36. Every paginated response reports whether more results remain, and carries the
    cursor a caller resumes from, so a caller distinguishes the end of a result
    set from a page that happens to be short and never constructs a position
    itself. No paginated return type carries either signal today. **[New]**

##### Chain and node metadata

What an integrator needs to know about the node it is talking to: where the
chain ends, which network this is, and how far back the data goes
([Appendix: Blockchain API and SDK Ecosystem, sections 1.3 and 1.6](../appendix/blockchain-api-sdk-ecosystem.md#13-identify-the-network-or-chain)).

- **Ethereum**: `eth_blockNumber` and `eth_chainId`.
- **Solana**: `getSlot` and `getGenesisHash`.
- **Bitcoin**: `getblockcount` and `getblockchaininfo`.
- **Zcash**: `getbestblockhash` and `getblockcount` for the tip,
  `getblockchaininfo` for the network.

**`query_block(block_id) -> PointerResult<FfiBlockOpt, OperationStatus>`**

Returns the block at a given identifier: its header, its full transaction body,
and its bedrock status.

37. The query returns the stored block for a known identifier and an absent
    `FfiBlockOpt` for one the indexer does not hold, keeping the two distinct
    from a backend failure. **[Ready]**
38. The returned header carries the block identifier, previous block hash, own
    hash, timestamp, and signature, and the body carries every transaction in
    the block. **[Ready]**

**`query_block_by_hash(hash) -> PointerResult<FfiBlockOpt, OperationStatus>`**

Returns the same block record, resolved by block hash rather than by height. The
store already holds the hash as a secondary index onto the height
(`lez/storage/src/indexer/read_once.rs:68-71`).

39. The query returns the same record `query_block` returns for the
    corresponding height, and an absent `FfiBlockOpt` for a hash no indexed
    block carries. A hash resolves to one block or to none, where a height
    resolves to whichever block currently occupies it. **[Ready]**

**`query_last_block() -> LastBlockIdResult`**

Returns the identifier of the indexer's last finalised block, inline, with no
allocation to free.

40. The query returns the last finalised block identifier, and reports an empty
    chain as an outcome distinct from an error. **[Ready]**

**`query_chain_tip() -> PointerResult<FfiChainTip, OperationStatus>`**

Returns the tip as one record rather than as a height the caller must then
resolve.

41. The FFI exposes the chain tip as a single call returning the tip block's
    height together with its hash and timestamp, so learning about the tip does
    not cost a second call and a caller knows which block holds the position.
    No such record exists below the FFI: `getLastFinalizedBlockId` returns a
    height alone, and the header fields come from a second read. **[New]**

**`query_network_identity() -> PointerResult<FfiNetworkIdentity, OperationStatus>`**

Returns what the indexer is reading: which zone, and which Logos Blockchain
chain that zone settles to.

42. The call returns the zone identifier the indexer is reading, so an
    application can confirm which zone it is connected to. It is read from the
    indexer's own channel configuration (`lez/indexer/core/src/config.rs:30`),
    not from the sequencer's `getChannelId`. **[Ready, not exposed]**
43. The same call returns the chain identifier of the Logos Blockchain the zone
    settles to. A zone identifier alone does not distinguish the same zone
    running against different L1 networks, which is the case an integrator
    connecting to the wrong network hits first, and the two are answered
    together so a caller cannot check one and assume the other. **[New]**
44. The chain identifier is the one inscribed in the L1 genesis block as a
    Cryptarchia parameter, a bounded UTF-8 string such as `logos-chain-1`, and
    not a value the indexer is configured with independently: a configured
    string would agree with whatever an operator typed rather than with the
    chain the node is settling to. It is inscribed and read at ledger
    initialisation but served by no L1 route
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.3](../appendix/blockchain-api-sdk-ecosystem.md#13-identify-the-network-or-chain)),
    and the indexer holds only an endpoint for its Bedrock connection
    (`lez/indexer/core/src/config.rs:19-29`), so obtaining it is work outside
    the indexer. A proposal states how it reaches the value. **[New]**
45. Where the chain identifier cannot be obtained, the call reports it as
    unavailable rather than omitting it or returning a placeholder, so a caller
    can tell an unidentified chain from an unasked question. **[New]**

**`query_program_ids() -> PointerResult<FfiVec<FfiProgramEntry>, OperationStatus>`**

Returns the programs the indexer has observed deployed, each with its name and
program identifier.

46. The FFI exposes the deployed programs as name and identifier pairs, derived
    from the `ProgramDeployment` transactions the indexer has ingested
    (`lez/indexer/service/protocol/src/lib.rs:283`), not from the sequencer's
    `getProgramIds`. An integrator decoding account data needs to know which
    program owns an account. **[New]**

**`query_retention_floor() -> PointerResult<FfiRetentionFloor, OperationStatus>`**

Returns the earliest block the indexer can still answer for, per read kind.

47. The FFI exposes the indexer's retention floor: the earliest block for which
    account state can be read at a pinned block identifier, and the earliest
    block for which a transaction can be retrieved. A deployment that retains
    everything reports genesis and one bounded by Reliability #7 reports a
    moving value, and a caller learns which by asking rather than by
    discovering it through a failed read. **[New]**

##### Surface-wide obligations

These bind every function above rather than adding one
([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).

- **Ethereum**: JSON-RPC error codes, and OpenRPC as the published contract.
- **Solana**: `getVersion` and the documented schema conventions.
- **Bitcoin**: numeric JSON-RPC error codes, with no published machine-readable
  contract.
- **Zcash**: the Bitcoin-inherited numeric error codes, with no Zcash-specific
  taxonomy published.

The module parity requirement has no ecosystem analogue, because no surveyed
chain interposes a plugin layer between its API and its consumers.

**`query_schema() -> *mut c_char`**

Returns a machine-readable description of the exported surface.

48. The FFI exposes a machine-readable description of its own surface, covering
    every exported function, its parameters, its return type, and its error
    codes. The existing `getSchema` describes the block type only and is not
    exported
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**

Every function defined above is further bound by the following.

49. Every function above is exposed through `lez_indexer_module` with the same
    semantics, including the not-found and error distinction required by
    Functionality #50. No capability reaching the FFI stops at the module
    boundary. **[New]**
50. The FFI and the module signal not-found, invalid-argument, and backend
    failure as three distinguishable outcomes on every query. The module
    currently flattens not-found and failure into an empty string
    ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
    **[New]**
51. Errors carry an application code from a documented, stable code space, a
    category, and a retryability signal. Two failure causes that require
    different caller recovery do not share a code. The current implementation
    uses the stock JSON-RPC `InternalError` code with free text
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).
    **[New]**
52. Every heap-allocating return has a documented matching free function, and
    calling it releases every allocation the return holds. **[Ready]**

#### Usability

1. Provide a Logos module surface (`lez_indexer_module`) covering every function
   in Functionality, usable to build Logos modules for reading LEZ state,
   following the chain, and tracking deposits.
2. Provide a CLI that covers core functionality: read an account at current and
   at a pinned block, read a batch of accounts, read a block, read a
   transaction, read a transaction's effects, query a
   transaction's status, list an account's transactions with ordering and
   pagination, and follow the chain tip. The CLI may have fewer features than
   the FFI but must support all essential operations.
3. Provide a worked deposit-tracking example, in the CLI or as a documented
   reference consumer, that follows the chain from a chosen start block, detects
   credits to a supplied set of public accounts, and reports each credit with
   its transaction hash, amount, block, and status level.
4. Document which state is readable and which is not for privacy-preserving
   transactions, and state that deposits into private accounts are not trackable
   from indexer data without the viewing key.
5. Return clear, actionable error messages for every failure mode, each mapped
   to the code space required by Functionality #51.
6. Document the semantics of every pagination parameter, including the
   exclusivity of the block bound, what a cursor guarantees across ingestion,
   and the behaviour when new data lands during a walk.

#### Reliability

1. A pinned account read at a given block identifier returns the same value on
   every call, however far ingestion has advanced, and returns the hash of the
   block it answered against so a caller can tell that a repeated read resolved
   to the same block. **[Ready, not exposed]**
2. A batch account read pinned to a block identifier returns a set of accounts
   consistent with one another as at that block, not a mixture of values read at
   different points during concurrent ingestion. **[New]**
3. Pagination cursors remain valid across ingestion: a caller walking blocks or
   an account's transactions by cursor neither skips nor repeats an entry that
   existed when the walk began.
4. A subscription consumer that disconnects and reconnects with its last
   processed block identifier receives every block after that position, with no
   gap and no assumption that the consumer was connected.
5. Every read reports the indexed tip it was served against, so a caller can
   detect that it read from a stalled or lagging indexer.
6. A stalled indexer is reported as stalled by the status surface rather than
   serving stale reads silently.
7. Storage growth is bounded by something the deployment controls rather than by
   the length of the chain. The indexer runs both as infrastructure an operator
   provisions and inside Basecamp on an end user's machine, where no one is
   watching a disk and no one can add one, so growth that a server operator
   would size for is a defect on a desktop. A deployment can put a ceiling on
   what the indexer retains, and reaching that ceiling degrades what the
   retention floor reports rather than failing reads or exhausting the disk.

#### Performance

1. Document the latency of every exported function, measured rather than
   estimated, against a stated chain size and transaction density.
2. Where a function's cost varies with something the caller controls or
   observes, document what it varies with and report the worst case as well as
   the typical one. A read of a past block, a batch sized by the caller, and a
   page bounded by a limit each cost more than a read at the tip, and a caller
   choosing between them needs to know by how much.
3. A batch account read of N accounts costs materially less than N single reads,
   and the improvement is measured and documented for representative N.
4. Document the storage cost of persisting per-transaction effects: bytes per
   transaction, and projected growth against a stated block rate and transaction
   density.
5. Benchmarks are reproducible from the test suite and run against a LEZ
   devnet/testnet indexer.

#### Supportability

This RFP defines the FFI. `lez_indexer_module` is what exposes it to Logos Core:
the module links `libindexer_ffi` and includes the generated `indexer_ffi.h`, so
the two ship together and a function reaches an application only when both carry
it. "The FFI and module" below means the Rust crate at `lez/indexer/ffi/` and
the module that links it.

1. Both are built and tested against a LEZ devnet and testnet sequencer.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   with an indexer ingesting from it, and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability, and Performance
   has at least one corresponding test.
5. Tests cover, at minimum: a read of an identifier the state holds no record
   for; a pinned read at a block before and after a state change; a batch
   read spanning recorded and unrecorded accounts; effects for a public transaction
   and for the public leg of a privacy-preserving transaction; membership proofs
   for a commitment the set holds and one it does not; a transaction status at
   each documented level; and a subscription resumed from a stored position
   across a disconnection.
6. A README documents end-to-end usage: building the FFI and the module that
   links it, the configuration and storage directory `start_indexer` requires,
   and step-by-step instructions for every operation via the CLI.
7. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the FFI and module, covering the developer integration journey for
   reading state, tracking deposits, and confirming transactions.
8. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator journey.
9. Provide a get started guide that takes a developer from an empty repository
   to a working Logos Core module consuming the indexer FFI, covering linking
   `libindexer_ffi`, including the generated header, reading LEZ state through
   it, and running the result against a devnet. The README required above
   documents building and operating what this RFP delivers; this documents
   building something else on top of it.
10. Provide a full API reference for the indexer API, published per version in
    the shape the Logos Storage module's reference takes at
    [docs.logos.co](https://docs.logos.co), covering every exported function,
    its parameters, its return type, and its error codes. The reference is
    generated from the machine-readable description required by Functionality
    #48 rather than maintained by hand, so it cannot drift from the header it
    describes.

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
5. Retention policies beyond a single ceiling, such as keeping transaction
   history for accounts a deployment cares about while discarding the rest, so
   a desktop deployment holds what its user needs rather than the most recent
   window of everything.

### Out of Scope

The following are explicitly excluded from this RFP:

- **Wallet and key management.** Key handling, derivation, watch-only address
  derivation, viewing keys, and signing belong to the LEZ wallet FFI, the second
  of the six deliverables.
- **Transaction construction and submission.** Building, signing, and submitting
  transactions run through the wallet path (`wallet_ffi` and `lez_core`). The
  indexer stack performs no writes and this RFP does not change that.
- **Simulation, and the cost estimate that comes with it.** Executing an
  unsubmitted transaction to see what it would do belongs to the wallet FFI, for
  both public and privacy-preserving transactions, for the reason given in the
  Design Rationale. Nothing in this RFP executes a transaction a caller
  supplies.
- **The JSON-RPC proxy and the language SDKs.**
  [logos-co/ecosystem#220](https://github.com/logos-co/ecosystem/issues/220)
  covers the transport, the wallet SDK, and the indexer SDK. Further transport
  bindings such as gRPC and GraphQL are
  [logos-co/ecosystem#222](https://github.com/logos-co/ecosystem/issues/222).
  This RFP defines what those consume, not how it is transported or wrapped.
- **Reaching the sequencer directly.** The FFI defined in this RFP is to be
  solely provided by the indexer module. A consumer never reaches the sequencer,
  whose interface is internal to LEZ and is not an API this RFP or any other in
  this set defines. Every read an integrator needs is required of the indexer
  above, whichever component holds the data today.
- **Transaction submission.** The indexer is a read-only follower and this RFP
  defines a read API. Submission stays on the sequencer's `sendTransaction`.
- **The program event system.** Events already exist end to end, from
  `ProgramOutput.events` through indexer capture to `getEvents`,
  `subscribeToEvents`, and `query_events` on the FFI. Nothing here changes them.
  The requirements below neither depend on events nor duplicate them, for the
  two reasons given in the Design Rationale: events do not cover private
  transactions, and their availability depends on how an operator configured the
  indexer's event filter.
- **Tracking deposits into private accounts.** Private post-states are encrypted
  and readable only by the viewing key holder, so this is foreclosed by the
  design of the chain rather than by the scope of this RFP.

## ⚠ Platform Dependencies

Every capability required above is either already present on the indexer RPC and
unexported, already computed by the indexer and discarded, or derivable from
data the indexer store already holds, with one exception.

The exception is the L1 chain identifier `query_network_identity` returns. It
exists, inscribed in the Logos Blockchain genesis block and read at ledger
initialisation, but no L1 route serves it and the indexer's Bedrock
configuration carries only an endpoint. Reaching it therefore depends on Logos
Blockchain exposing it, or on the zone obtaining it at initialisation and
retaining it. The requirement that an unobtainable value be reported as
unavailable keeps the rest of the surface deliverable while that is
outstanding.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md): as-built
  inventory of the LEZ indexer FFI and RPC, the sequencer RPC, the wallet FFI
  and `lez_core` module, and the L1 bindings, routes, and module
- [Appendix: Blockchain API and SDK Ecosystem](../appendix/blockchain-api-sdk-ecosystem.md):
  34 API functions across nine established chains, with transports, SDK
  languages, response shapes, and per-function gap notes for LEZ
- [logos-execution-zone](https://github.com/logos-blockchain/logos-execution-zone):
  the LEZ sequencer, indexer, and wallet
- [lez-indexer-module](https://github.com/logos-blockchain/lez-indexer-module):
  the Logos Core module wrapping the indexer FFI
- [Logos glossary](https://docs.logos.co/get-started/glossary): zone, channel,
  Bedrock, Basecamp, module, viewing key, and note
- [LEZ documentation](https://docs.logos.co/lez): the sequencer and indexer
  roles, and the separation of public and private state
- [Logos Documentation](https://github.com/logos-co/logos-docs)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
