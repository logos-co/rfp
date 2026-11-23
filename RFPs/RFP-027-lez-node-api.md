---
id: RFP-027
title: LEZ Node API
tier: L
status: open
category: Developer Tooling & Infrastructure
dependencies: []
---

# RFP-027 — LEZ Node API

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> This specification may also refer to other RFPs, LPs or their specifications.
> Such references are made for the purpose of identifying technical
> requirements, capabilities, dependencies or criteria applicable to a
> Submission under this RFP, including components that may need to be developed
> or incorporated as part of that Submission.
>
> Such references do not, in themselves, require or direct an Applicant to use,
> extend, modify or otherwise rely on any previous code, implementation or
> Submission associated with the referenced RFP, LP or specification. Where such
> references are made, it is for the Applicant to determine how best to develop
> or incorporate the relevant components to satisfy the requirements of the
> current RFP, whether to develop them independently or otherwise.
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
> does not make any representation, provide any advice or assume any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

This RFP invites proposals for the LEZ node API: the set of functions an
integrator needs to follow the chain, track deposits, confirm transactions, and
submit a transaction it has already signed, specified as exported functions on
the LEZ node FFI and surfaced through
[`lez_indexer_module`](https://github.com/logos-blockchain/lez-indexer-module).
Both are existing Logos-maintained repositories, and this specification is
designated for integration into them: the work is contributed to
`logos-execution-zone` and `lez-indexer-module` through their usual review and
merge process. This is the equivalent of the `eth` namespace on Ethereum's
JSON-RPC: it answers what happened and what the current state is, and it accepts
a signed transaction for the network, holding no keys and signing nothing.

### Target architecture

This RFP assumes the target architecture described in
[Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos),
parts of which are still planned.

- **Each chain is served by a wallet module and a node module.** The wallet
  module holds keys, derives, signs, and proves, and runs where the keys are.
  The node module follows the chain, serves its state, and relays the
  transactions wallets produce. A wallet reaches its node only through the node
  API, which this RFP defines for LEZ. Everything on the wallet side belongs to
  the LEZ wallet API, a separate component outside this RFP. Today `lez_core`
  combines the LEZ wallet with a direct connection to a sequencer; in the
  target, `lez_indexer_module` is the LEZ node module, and the wallet reaches
  the chain through it alone.
- **Every LEZ node is an indexer.** It obtains the finalised LEZ state from a
  Logos Blockchain node and serves it. Block, account, and history reads
  therefore answer for blocks final on the Logos Blockchain, and only the
  transaction status levels short of final (#22) and the pending set (#34) speak
  about what is not yet final.
- **A sequencer is a LEZ node in sequencer mode.** It serves the same API, and
  also holds the pending set it builds blocks from, produces blocks, and
  inscribes them with a Logos Blockchain wallet. This RFP defines every function
  once. Where a function or an outcome is available only in sequencer mode, or
  means something different there, the requirement says so, and a node outside
  sequencer mode reports it as not available in its mode (#83).
- **LEZ nodes talk to each other peer to peer.** Relaying a submitted
  transaction to the nodes that sequence, gossiping it, and propagating blocks
  run over the LEZ peer-to-peer network and are not part of this API (see
  [Out of Scope](#out-of-scope)).
- **The API is the module's contract.** As every Logos module does,
  `lez_indexer_module` publishes its contract as a CDDL schema (#80), and
  callers use clients generated from it. The same contract serves a caller in
  the same runtime and one on another device or in an integrator's
  infrastructure, reached through Logos Core peering, which carries calls and
  events alike. The module serves the node through the indexer FFI, so the
  requirements below are stated on FFI functions, and a capability reaches an
  application only when the FFI and the module both carry it (#84). See
  [Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md) for the
  as-built inventory of every layer.

Today the indexer and the sequencer run as separate services, and the indexer
joins no peer-to-peer network. Platform Dependencies records what the target
needs that does not exist yet.

### Who calls it, and for what

One LEZ node serves several independent wallets at once. Wallets belonging to
different users, in different applications or on other Logos Core runtimes, read
from the same node and submit through it, and none of them operates it. The
surface therefore separates the functions any wallet calls from the functions
that operate the node, and each function is granted to callers individually.

Applications on LEZ are built on Basecamp and Logos Core, on mobile, desktop and
server. An existing application can also add LEZ support without re-architecting
itself, by hosting Logos Core and bundling the modules it needs, and the node
API is specified so that it serves that case as well.

The parties that integrate a chain before it is usable in practice are wallets,
centralised exchanges, custodians, payment gateways, data and price aggregators,
node and RPC providers, fiat on and off ramps, bridges, and tax and accounting
providers. Of those, the centralised exchange is used as the reference profile
for reading and confirming: read chain state, track deposits credited to
accounts it controls, and confirm that a transaction reached a level of
certainty it is willing to act on. It is the strictest reader of the set. The
wallet is the reference profile for what an exchange never asks: the membership
proofs it spends with, the pending set it checks while proving, and the
publications and encrypted notes it receives through.

The scope is therefore the node's reads of chain state, transaction effects and
status, the pending set, subscriptions, the reads a wallet spends and receives
through, submission of a signed transaction, and control over who may call which
function. The LEZ node API is one of the components the Logos stack exposes for
integration, alongside the LEZ wallet API, the Logos Blockchain node and wallet
APIs, and the Logos Core host libraries, which let an application host Logos
Core. The shapes an integration takes and what embedding a node costs are
described in
[Appendix: Integrating the Logos Technology Stack](../appendix/integrating-logos-technology-stack.md).

Wallet and key management, transaction construction and signing, communication
between LEZ nodes, the Logos Blockchain APIs, and hosting Logos Core in an
application are separate components and are outside the scope of this RFP.

## 🔥 Why This Matters

An integrator adding a chain reaches it through a node API, and expects that API
to answer the same questions it answers on the chains the integrator already
supports. The requirements below are therefore drawn from what those integrators
rely on elsewhere. Nine established chains were surveyed for that purpose. LEZ
does not currently meet several of the norms: there are no per-transaction
effects; no confirmation level; and no subscription reaches the FFI, which is
the boundary an integrator can actually call. Some norms this RFP declines
rather than fills: simulation, for the reason given in the Design Rationale, and
the functions that go with constructing a transaction rather than reading the
chain, which [Out of Scope](#out-of-scope) lists one by one. See
[Appendix: Blockchain API and SDK Ecosystem, section 4](../appendix/blockchain-api-sdk-ecosystem.md#4-gap-summary).

Most of the missing capability already exists one layer below the FFI, as the
readiness markers below record, so the work is largely exposure rather than
invention.

## 🏗 Design Rationale

### The target is LEZ testnet 0.4

The node API is delivered for LEZ testnet 0.4 (Supportability #1). The readiness
markers below were read from the LEZ default branch at `334d0fc29`, after
`v0.4.0-rc3`, and they describe the starting point for the work rather than a
constraint on what the surface may express. File references below are to that
commit.

Fees are already in force there. A public transaction declares a payer, a gas
limit, a tip, and a cap on the fee
(`lez/indexer/service/protocol/src/lib.rs:252-268`). A privacy-preserving
transaction pays in one of two ways: from a reserve its proof commits to, naming
where a surplus is refunded (`lez/indexer/service/protocol/src/lib.rs:334-346`),
or through a fee declaration signed by one of its public signers, settled as a
public transaction's is (`lez/indexer/service/protocol/src/lib.rs:362-363`,
`lez/chain_state/src/classify.rs:171-183`). Gas is metered in two dimensions,
execution and storage (`lez/programs/fee/core/src/lib.rs:24-29`), and storage is
charged on a transaction's serialized size, a privacy-preserving transaction's
proof being counted at a fixed length (`lez/common/src/transaction.rs:38-57`).
Cost is therefore something a caller estimates, reports, and budgets against.
The fee each transaction paid is computed at ingest and then discarded; only a
block total survives.

[Functionality](#functionality) #5 requires the fee to be reported separately
from the balance changes, so a caller reads it rather than reconstructing it by
arithmetic, and #6 requires the gas consumed alongside it.

### Simulation belongs to the wallet, not to the node

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
plaintext a simulation would need, and there is nothing for the node to simulate
before the wallet has already done the work. Zcash exposes no simulation of its
shielded leg either, for the same reason, though it exposes none of its
transparent leg, which has no such obstacle.

Public execution could be simulated by the node, but siting it there would split
one capability across two components and answer against the node's state rather
than the state the caller is building on, its own transactions not yet included
among it. It would also show the node every transaction a caller considered, not
only those it submitted, which a node several wallets share should not see
(Privacy #5, Privacy #6). The wallet already holds the executor for the private
path and can read whatever state it needs through the queries below, so it can
answer for both kinds of transaction, against a state it chose, at the moment it
is constructing the transaction. That is also what simulation is for elsewhere:
Stellar returns the transaction data and the minimum resource fee that the
caller copies back into the transaction before submitting, which makes
simulation a construction step rather than a read. Construction is out of scope
here and belongs to the LEZ wallet API, defined separately.
[Out of Scope](#out-of-scope) records the exclusion.

The LEZ wallet already works this way. Before it sends a public transaction it
simulates it locally against the actor states and program loader state it has
read, measures the gas the run consumed to set the declaration's gas limit, and
refuses a send whose simulation reverts unless the caller allows it
(`lez/wallet/src/lib.rs:228-282`, `lez/wallet/src/lib.rs:2286-2296`,
`lez/wallet/src/lib.rs:2640-2717`). What it needs from a node for that is reads:
the states the transaction touches, which the account reads below serve, and the
current fee market, which it prices the declaration against and reads from the
sequencer service's `getFeeState` today (`lez/wallet/src/lib.rs:348-350`).
[Functionality](#functionality) #67 and #68 require the node to serve the fee
market, so a wallet simulating on its own side reaches every input it needs
through the node API.

### Inspiration from BDK: Bitcoin Development Kit

The [Bitcoin Development Kit](https://bitcoindevkit.org) is the closest
precedent for the split this RFP sits on one side of: a wallet that holds the
keys, and a chain client that holds none. Everything below was read at `bdk-ffi`
`3.1.0-alpha.0`.

BDK is what a wallet developer uses instead of writing wallet logic against a
node. It is layered. At the bottom sits
[`rust-bitcoin`](https://github.com/rust-bitcoin/rust-bitcoin), types and
consensus encoding only: it is `no_std` and contains no networking. Above it,
[`bdk_wallet`](https://github.com/bitcoindevkit/bdk_wallet) holds the wallet
proper, descriptors, key derivation, coin selection, transaction building and
signing, and the wallet's own view of the chain. Beside it sit four
interchangeable chain clients, each a separate crate: `bdk_esplora` for HTTP,
`bdk_electrum` for Electrum servers, `bdk_bitcoind_rpc` for Bitcoin Core's
JSON-RPC, and `bdk_kyoto` for P2P compact block filters.

[`bdk-ffi`](https://github.com/bitcoindevkit/bdk-ffi) then wraps the wallet and
those clients for consumers that are not writing Rust. It is its own crate,
depending on the wallet crate and re-exposing it across a UniFFI boundary rather
than re-exporting it, and the Kotlin, Swift and Python packages are generated
against it. A Kotlin consumer never sees Rust: it adds one dependency,
`bdk-android`, whose entire Kotlin source tree is a build artefact regenerated
from the FFI crate.

That arrangement answers two questions about the split between a wallet and a
node: how the pieces are packaged, and how the wallet and a client relate to
each other at runtime.

The first is packaging. Wallet and chain access ship as one crate, one UniFFI
namespace, and one native library: `lib.rs` declares `mod wallet` beside
`mod esplora`, `mod electrum`, and `mod kyoto`, then calls
`uniffi::setup_scaffolding!("bdk")` once. Those modules organise the Rust source
rather than the exposed surface, and they collapse at the FFI boundary: the
Android test that exercises `Wallet` and `EsploraClient` together imports
nothing from BDK at all, because both arrive in one flat package. What separates
the wallet functions from each client's functions is the object they hang off,
not a namespace. Every export is a method on a type and none is a free function,
so `Wallet` carries the wallet operations and `EsploraClient` the chain reads,
and a caller disambiguates by naming the object. The cost of the single artefact
is that no Cargo feature gates those backends, so every consumer links all three
whether it uses one or not.

The second is control flow, which is what keeps those four clients
interchangeable. The wallet holds no client and performs no network I/O. It
exposes no method returning a network error, and chain data reaches it through
one door: the consumer calls an `apply_*` method with data the consumer fetched.
Sync is three steps the integrator writes, not one call the wallet makes:

```
wallet.start_full_scan()   -> FullScanRequest   (a value)
client.full_scan(request)  -> Update            (a value)
wallet.apply_update(update)
```

The seam is a pair of plain values rather than a trait the wallet defines and a
client implements. The dependency direction never inverts: the chain modules
import `Update` from the wallet's types, and the wallet imports nothing from
them. That is what lets three transports as different as HTTP, Electrum, and P2P
compact block filters sit behind one import without the wallet knowing which is
in use, and it survives projection onto Kotlin and Swift, which a trait does not
do cleanly.

The Rust-native path is a different shape again, so one client interface does
not serve every consumer. Where the HTTP and Electrum clients use the request
and apply pair above, `bdk_bitcoind_rpc` talks to a node over JSON-RPC through a
long-lived `Emitter` the consumer drives as a pull loop: the wallet's checkpoint
and unconfirmed set are injected once at construction, `next_block()` is called
until it returns nothing, and each block is applied individually with
`apply_block_connected_to`. There is no request to build. The emitter holds the
reorg state and the mempool snapshot, so that state sits in the client rather
than the wallet. Two transports against the same wallet therefore present two
different consumer flows, and the wallet accommodates both only because it
exposes several `apply_*` entry points rather than one.

Two consequences for the node API. A wallet reaches its node only through the
node API, and the node API is an interface more than one module can implement,
as four clients sit behind one BDK wallet. The functions below are therefore
specified by what they answer, whichever component holds the data today, and
nothing in them assumes where the node runs. And because the three-step sequence
is verbose enough that integrators tend to wrap it themselves, a convenience
that performs a sync against a supplied node is worth offering rather than
leaving every consumer to write it: the `lez_core` module ships one. That
convenience belongs to the LEZ wallet API.

### What a node can answer about a privacy-preserving transaction

**The constraint.** A public account's state lives in the replicated state
machine, so a node holds its plaintext. A private account's state exists
on-chain only as a commitment, and its plaintext travels inside the transaction
encrypted to the account's viewing key. A node holds no viewing keys.

A privacy-preserving transaction is not opaque in full. A
`PrivacyPreservingMessage` carries, in plaintext, the deliveries that cross
between its private execution and public state in either direction, an optional
public root call, the public nonces it consumes, its validity windows, and its
fee terms (`lez/indexer/service/protocol/src/lib.rs:298-364`). The node executes
that public part itself
(`lee/state_machine/src/validated_state_diff/mod.rs:385-410`), so the public
leg's effects are as readable as a public transaction's, but only from
execution: the body carries the inputs to the public part, not its post-states.
The fee is paid in the clear either way: a fee reserve is one of those
crossings, and it names the address it was paid from
(`lez/chain_state/src/classify.rs:113-128`), and a declared fee names a payer
that signs the transaction.

The private part carries four kinds of encrypted note. Each private action
carries a `nullifier`, a `commitment`, the commitment set `root` it was proven
against, and an `encrypted_post_state`; each async message carries a commitment
and a note; each recovery binding carries a private address and a note; and the
fee fallback carries a private action of its own
(`lez/indexer/service/protocol/src/lib.rs:323-395`). A node can report that a
private action occurred, prove a commitment's membership, and serve every
ciphertext. What it cannot read is the plaintext those ciphertexts hold: which
private account an action touched, and the state it now carries, its nonce and
its actor states, the native balance among them
(`lez/indexer/service/protocol/src/lib.rs:100-127`).

A public-to-private send sits between the two. It is published as a clear
message whose envelope names the sender, the receiving private address, and the
message in plaintext (`lee/state_machine/core/src/program/mod.rs:314-335`), so
the amount and the receiving address of a public-to-private send are readable,
while who controls that address and what it holds afterwards are not. A
private-to-private send is published sealed, and reveals neither.

Two limits bound this. Decryption requires the viewing key, which belongs to the
wallet. And a node cannot link a private action's input to its output: the
protocol records that a private action's commitment is not necessarily connected
in content to its nullifier (`lez/indexer/service/protocol/src/lib.rs:327-329`).

**Consequences for the API.** The surface serves the wallet the material it
needs to interpret private state itself, and serves other consumers the public
leg. Commitments, nullifiers, roots, and ciphertext are returned rather than
elided, and each read says which side of the boundary it answers for, so a
consumer is not left to infer the boundary from the shape of a response.

[Functionality](#functionality) #2 requires the returned transaction to
discriminate public and privacy-preserving bodies, so a consumer knows which it
is holding. #4 scopes the balance deltas the effects query returns to public
transactions and the public leg of privacy-preserving ones, which is what an
integrator crediting deposits reads: a deshield into a public account is
reported, and a transfer between private accounts is not. Because the public leg
of a privacy-preserving transaction is readable only by executing it, #3
persisting what ingest computed is what makes that leg reportable at all.

The wallet is the other consumer, and there the work is on this surface rather
than on the capability. #19 and #20 require the membership proofs and the root
they are proven against, which a wallet needs to spend. The publication listing
and recovery-binding reads under Private message delivery are what a wallet
needs to receive. The sequencer service answers all of these today, and its own
RPC marks them for removal once the wallet reads from the indexer
(`lez/sequencer/service/rpc/src/lib.rs:64-65`), while the state the indexer
already holds carries the data (`lee/state_machine/src/state/mod.rs:363-430`).
What is new is serving them from the node. #51 requires the oldest-first
ordering a wallet walking forward from its last processed block relies on to
decrypt what it can.

### Reading with a viewing key alone

**Who holds only a viewing key.** Several consumers need to read a private
account without being able to move what it holds: an auditor, an accountant or
tax provider, a custodian monitoring funds whose spending keys are held offline,
and a member of a shared private account who is given sight of it but not
control. Each holds a viewing key and nothing else. Zcash serves the same
consumers by handing the node the viewing key, `z_importviewingkey`, which
leaves the node's operator able to read everything the key can. The node API
here takes the opposite position: the node holds no viewing keys (see the
constraint above), so a viewing key consumer decrypts on its own side, and the
node's job is to serve it the material to decrypt.

**What a viewing key opens.** A private account has three secrets: an
authorisation key, a nullifier key derived from it, and a viewing key
(`lee/key_protocol/src/key_management/secret_holders.rs:104-140`,
`lee/state_machine/core/src/nullifier.rs:76-87`). Every encrypted note is
encrypted under a key derived from a secret shared with the recipient's viewing
key and from the action's nullifier, which is public
(`lee/state_machine/core/src/encryption/mod.rs:91-150`), and a regular private
account's identifier follows from its public keys alone
(`lee/state_machine/core/src/nullifier.rs:29-47`). So a viewing key on its own
opens every note addressed to it, and the consumer can check that the
transaction commits the state it decrypted.

**What it does not open.** Every nullifier, whether it spends an account state
or a received message, is keyed by the nullifier key
(`lee/state_machine/core/src/nullifier.rs:126-158`). The LEZ wallet follows its
accounts by computing each one's next nullifier in advance and watching for it
(`lez/wallet/src/lib.rs:2956-2979`), which a viewing key holder cannot do. Nor
can the nullifier key be handed out as a weaker credential: a witness holding it
alone can update a regular private account, and it is the whole authority over a
private PDA (`lee/state_machine/core/src/circuit_io.rs:290-310`). A viewing key
consumer therefore trial-decrypts: it opens every note in chain order and keeps
those that decrypt and that the transaction commits, which is how the wallet
already discovers accounts it does not yet follow
(`lez/wallet/src/storage/key_chain.rs:556-592`). What it cannot establish is
whether a note it opened has since been consumed, other than by seeing the state
that replaced it.

**Shared private accounts.** A shared private account, whether a regular account
or a private PDA, is keyed from a group master secret every controller holds,
from which each account's keys are derived
(`lee/key_protocol/src/key_management/group_key_holder.rs:38-160`). The master
secret derives spending keys, so it cannot be shared for viewing; giving a party
sight of a shared account means giving it the viewing key derived for that
account. From the node's side, a shared account's notes are indistinguishable
from any other private account's, and they are served the same way.

**Consequences for the API.** A viewing key consumer walks the chain in order
and needs every encrypted note, with the public values that decrypt and check
it, and nothing it cannot use: the proofs and witness sets a privacy-preserving
transaction carries are of no use to it. Zcash reached the same shape in the
compact blocks of [ZIP 307](https://zips.z.cash/zip-0307), which carry only what
a light client needs to detect payments to its keys, and which lightwalletd
streams by height range
([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).
[Functionality](#functionality) under Viewing key reads requires that walk, for
every private account kind, parameterised by position alone so the walk itself
reveals nothing about which accounts the consumer holds keys for. Defining how a
viewing key is exported, and whether a group of controllers should hold a
viewing secret of its own, belongs to the LEZ wallet API.

### One node, many wallets

**The node keeps nothing per wallet.** It holds no keys, and every read is an
answer about the chain, which is the same answer whoever asks. A wallet does not
register with the node, and the node does not need to know a wallet exists in
order to serve it. What is per caller is narrow: the subscriptions a caller
opened and the calls it has in flight. A walk part way through is carried by the
cursor or position the caller holds, not by the node. The requirements keep
those from reaching across callers, and keep the node's lifecycle out of every
wallet's hands.

**Access follows the caller and the function, not a role.** Logos Core attaches
a caller identity to a call it dispatches. The receiving runtime writes that
identity, it is not a parameter of the method, and a module reads it. It names
the runtime itself or a named module, a shell and a UI plugin arriving as named
modules, and work in development adds an operator and a consumer on another
runtime. The deployer's access policy names, per target module, which callers
may reach it, and its per method form narrows a caller to a list of methods
([logos-liblogos#232](https://github.com/logos-co/logos-liblogos/pull/232),
[logos-capability-module#37](https://github.com/logos-co/logos-capability-module/pull/37)).
Logos Core defines no user or administrator class.

This RFP therefore proposes two sets of functions rather than two kinds of
account. The **administrative set** is the functions that operate the node:
starting it, stopping it, and resetting its storage. The **user set** is every
other function [Functionality](#functionality) defines, which is what a wallet
or integrator calls. A deployer grants either set, or any single function, to
the callers it chooses. #89 fixes the assignment and #90 requires each function
to be individually grantable.

**The administrative set is closed unless granted.** Enforcement of the access
policy is off unless the deployer turns it on, and a policy in force may admit a
caller to every method of the module, so an administrative function cannot rely
on the policy to protect it. #91 requires the module to refuse an administrative
call itself, on the caller identity, unless the policy grants that caller the
function by name or the caller is the runtime, its shell, or an operator of that
runtime. A caller the runtime cannot identify is refused.

**The boundary is the module, not the FFI.** A caller identity exists only on a
call that arrives through Logos Core. Whoever links `libindexer_ffi` and holds
the handle is the node's host and can call every exported function, so the
requirements below bind `lez_indexer_module`, which is the surface a wallet
reaches.

**Callers are no finer than Logos Core makes them.** Two wallets in two modules,
two UI plugins, or two runtimes are two callers. Two wallets behind one calling
module are one caller, and the node cannot tell them apart. Because the node
keeps nothing per wallet, it does not need to. [Out of Scope](#out-of-scope)
records that the node issues no credentials of its own.

### Retention is bounded and reported

A deployment bounds what the node retains (Reliability #7), and the node reports
what it holds rather than leaving a caller to discover it through a failed
request. Cosmos reports `earliest_block_height` and Bitcoin `pruneheight` for
that purpose, and XRPL reports what it retains as a range that may have gaps
([Appendix: Blockchain API and SDK Ecosystem, section 1.2](../appendix/blockchain-api-sdk-ecosystem.md#12-health-and-sync-status-check)).
EIP-4444, which bounds the history Ethereum clients keep, does so deliberately
so that callers seek old data elsewhere, which only works if a caller can tell
what a node no longer holds
([section 1.29](../appendix/blockchain-api-sdk-ecosystem.md#129-read-historical-state-at-a-past-version)).

[Functionality](#functionality) #69 requires what the node retains to be
exposed, and #70 requires a read below it to be answered as such, distinctly
from a read for data that never existed.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

Each requirement carries a readiness marker describing its starting point in the
current LEZ codebase. **Ready** means the capability is already exported by the
node FFI and only a specification, test, or documentation obligation remains.
**Ready, not exposed** means it exists in the indexer service, RPC, or store but
the FFI does not export it, so the work is exposure. **Computed, not persisted**
means the indexer derives the data during ingestion but neither stores nor
exposes it, so the work is persistence plus exposure. **New** means neither the
data nor the capability exists today. The markers indicate effort, not priority:
every hard requirement is required regardless of its marker. When not already
implemented, the function signatures below should be seen as suggestions; the
description of functionality underneath is the requirement.

The markers describe the FFI on the LEZ default branch. `lez_indexer_module` is
built against LEZ `v0.3.1` (`lez-indexer-module/flake.nix:14`), from before the
FFI's account read became optional and its account record moved to actor states,
so a marker of **Ready** means ready in the FFI, and the module reaches it only
once it is rebuilt against the testnet 0.4 FFI, which #84 requires.

Every signature below elides a first argument, the indexer handle returned by
`start_indexer`. See Surface-wide obligations.

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
2. The returned transaction discriminates public and privacy-preserving bodies,
   exposing for each the fields the wire type carries
   (`lez/indexer/service/protocol/src/lib.rs:215-395`). A program is deployed by
   a public transaction to the program loader rather than by a transaction kind
   of its own. **[Ready]**

**`query_transaction_effects(hash) -> PointerResult<FfiOption<FfiStateDiff>, OperationStatus>`**

Returns the state diff a transaction produced, read from storage rather than
recomputed on the caller's behalf.

3. The FFI exposes a per-transaction effects query that, given a transaction
   hash, returns the state diff that transaction produced: the accounts whose
   public state changed, their pre-state and post-state nonces and actor states,
   the new commitments, the new nullifiers, and the program invoked. The diff is
   the one the chain applied when the transaction settled, as the indexer
   computes it at ingest (`lez/chain_state/src/apply.rs:147-390`), not one
   recomputed for the caller. A charged transaction settles as separate diffs
   for its fee reserve, its action, and its refund, and each is reported with
   the state it applied to; a diff as computed today carries post-states only
   (`lee/state_machine/src/validated_state_diff/mod.rs:36-45`). **\[Computed,
   not persisted\]**
4. The effects query returns a balance delta per affected account, expressed as
   pre-state and post-state values, for public transactions and for the public
   leg of privacy-preserving transactions. The native balance is the account's
   actor state at the native token program
   (`lez/indexer/service/protocol/src/lib.rs:106-127`), and a token held through
   another program is that program's actor state, so the delta is reported per
   actor state rather than as a single balance. **[Computed, not persisted]**
5. The fee is reported separately from the balance changes the transaction
   itself caused, naming the account charged, the amount taken, and any amount
   refunded. An exchange reconciling a withdrawal has to tell what it sent from
   what it paid to send it, and a caller that has to reconstruct the fee by
   arithmetic on the remaining deltas is reimplementing the chain's fee rules. A
   public transaction declares a payer, a gas limit, a tip, and a cap on the
   fee; the payer need not be an account the transaction otherwise acts on, but
   it must sign (`lee/state_machine/src/fees.rs:34-52`). A privacy-preserving
   transaction pays either from a reserve its proof commits to, with any surplus
   refunded to a target it names, or through a fee declaration signed by one of
   its public signers, as a public transaction does, and the effects query
   reports which. The fee each transaction paid is computed when it settles and
   is not retained; only the block's total is. A fee-exempt system transaction
   is reported as exempt rather than as one that paid nothing. **\[Computed, not
   persisted\]**
6. The gas a transaction consumed is reported alongside the fee, in each
   dimension the chain meters, execution and storage
   (`lez/programs/fee/core/src/lib.rs:24-29`), so a caller comparing what it
   budgeted against what it paid can tell a transaction that ran long from one
   that paid a high price per unit. Both kinds of transaction are metered: a
   privacy-preserving transaction is charged a fixed verification cost, storage
   on its serialized size with its proof counted at a fixed length, and the
   metered execution of its public part
   (`lez/chain_state/src/apply.rs:513-542`). Only a fee-exempt system
   transaction meters nothing, and the API distinguishes it from a transaction
   that consumed zero. **[Computed, not persisted]**

**`query_events(from_block, to_block, tx_hash, program_account_id, selector) -> PointerResult<FfiVec<FfiEventRecord>, OperationStatus>`**

Returns the program events matching the filter. A non-null `tx_hash` makes the
call a point lookup and the block range is ignored; otherwise the range runs
from `from_block` to `to_block`, defaulting to the indexed tip.

7. The query returns each matching event record with its block identifier,
   transaction index, transaction hash, emitting program, selector, and data
   payload. **[Ready]**
8. A request the query cannot answer is reported as the outcome #85 and #87
   define for its case rather than as an empty result: a range spanning more
   than `MAX_EVENT_QUERY_BLOCK_SPAN` blocks, or inverted, as an invalid
   argument; a bound past the indexed tip as the lagging case; a range outside
   the node's event-filter history as data the node does not hold, as a read
   below the retention floor is; and a point lookup for a hash no indexed
   transaction carries as not found. Each returns `InvalidArgument` today
   (`lez/indexer/ffi/src/api/query.rs:507-527`). **[New]**
9. Events emitted by the public part of a privacy-preserving transaction are
   indexed as a public transaction's are
   (`lee/state_machine/src/validated_state_diff/mod.rs:585-620`), and execution
   inside the proof emits none a node can observe, so the query answers for
   public execution only. **[Ready]**

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

**`query_account(account_id) -> PointerResult<FfiOption<FfiAccount>, OperationStatus>`**

Returns the account record as it stands at the node's current state: its nonce,
and one actor state per program holding state on the account.

10. The query returns the account's nonce and its actor states, each keyed by
    the account identifier of the program it belongs to
    (`lez/indexer/service/protocol/src/lib.rs:100-110`). The native balance is
    the actor state at the native token program, and the API documents how to
    read it. **[Ready]**
11. An identifier the state holds no record for returns an absent `FfiOption`,
    distinct from a recorded account whose actor states are empty and from a
    backend failure. **[Ready]**

**`query_account_view(account_id, program_account_id, block_id) -> PointerResult<FfiOption<FfiAccount>, OperationStatus>`**

Returns an account's nonce and the actor state of one program, optionally pinned
to a block.

12. The FFI exposes a read of one program's actor state on an account, with the
    account's nonce, at the tip or at a pinned block, exporting the existing
    `getAccountView` and `getAccountViewAtBlock` indexer RPC methods
    (`lez/indexer/service/rpc/src/lib.rs:82-90`). Nothing bounds how many actor
    states an account holds or how large each is, and any party can write its
    own onto any account (`lez/indexer/service/protocol/src/lib.rs:136-146`), so
    a whole-account read is not a safe way to read one balance; this is.
    **[Ready, not exposed]**

**`query_account_summary(account_id) -> PointerResult<FfiOption<FfiAccountSummary>, OperationStatus>`**

Returns an account's nonce, its native balance, and the size of each actor state
rather than its bytes.

13. The FFI exposes an account summary: the nonce, the native balance, and the
    size of each actor state rather than its bytes, exporting the existing
    `getAccountSummary` indexer RPC method, so a caller learns which programs
    hold state on an account in a response whose size follows their count.
    **[Ready, not exposed]**
14. An identifier the state holds no record for returns an absent `FfiOption`,
    on the same terms as the account read (#11). The RPC method returns a
    default summary for it today (`lez/indexer/service/src/service.rs:152-161`),
    which a caller cannot tell from a recorded account that is empty. **[New]**

**`query_account_at_block(account_id, block_id) -> PointerResult<FfiOption<FfiAccount>, OperationStatus>`**

Returns the account record as it stood at a given block, rather than at the tip.

15. The FFI exposes an account read pinned to a block identifier, returning the
    account record as it stood at that block, on the same terms as the read at
    the tip. This exports the existing `getAccountAtBlock` indexer RPC method
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[Ready, not exposed]**
16. The pinned read returns the hash of the block it answered against, so a
    caller can match the answer to a block it already holds, and two answers
    from different nodes to the same chain. The node holds only blocks final on
    the Logos Blockchain (`lez/chain_state/src/zone_indexer.rs:54-86`), so an
    identifier it answers for names one block. **[New]**

**`query_accounts(account_ids, block_id) -> PointerResult<FfiVec<FfiOption<FfiAccount>>, OperationStatus>`**

Returns one result per requested account identifier, optionally pinned to a
block.

17. The FFI exposes a batch account read taking a list of account identifiers
    and returning a result per identifier in request order. An identifier the
    state holds no record for is reported absent in its entry rather than
    failing the call. **[New]**
18. The batch account read accepts an optional block identifier that pins every
    account in the batch to the same block, so a multi-account read is
    internally consistent under concurrent ingestion. **[New]**

##### Commitment membership proofs

Reading the inclusion proofs a caller needs to spend a private account state or
consume a received message, and the root they are proven against. Zcash is the
one surveyed chain with a comparable structure. It serves the state of its
commitment tree at a height, `z_gettreestate` on the node and a tree-state read
on lightwalletd, from which a light client keeps its own witnesses up to date
rather than asking for a proof of the notes it holds
([Appendix: Blockchain API and SDK Ecosystem, section 1.29](../appendix/blockchain-api-sdk-ecosystem.md#129-read-historical-state-at-a-past-version)).

**`query_commitment_proofs(commitments, message_positions) -> PointerResult<FfiCommitmentProofs, OperationStatus>`**

Returns a membership proof per requested commitment and per requested message
position, together with the commitment set root they are proven against.

19. The FFI returns, for a list of commitments and a list of message positions,
    a membership proof per entry in request order and the commitment set root
    those proofs are proven against. A commitment or position the set does not
    hold is reported as absent per entry rather than failing the call. This is
    the data the sequencer service's `getProofsAndRoot` returns today
    (`lez/sequencer/service/rpc/src/lib.rs:124-138`), which the wallet reaches
    the sequencer for (`lez/wallet/src/lib.rs:1269-1284`), and the state the
    indexer already holds answers it (`lee/state_machine/src/state/mod.rs:207`,
    `lee/state_machine/src/state/mod.rs:423-430`). **[Ready, not exposed]**
20. The proofs and the root returned by one call are consistent with one
    another: every proof verifies against the returned root, as those
    `getProofsAndRoot` returns do. **[Ready, not exposed]**
21. The API documents that this read names to the node the commitments and
    message positions the caller is about to spend, which is what a node shared
    by several wallets learns from it, and Privacy #7's list of disclosing calls
    includes it. Soft #7 asks for a way to spend that names none. **[New]**

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

22. The FFI exposes a per-transaction status query returning a level from a
    documented set that distinguishes at minimum: not known, pending, included
    in a block not yet final, dropped (#25), and final. Each level documents
    what a caller may conclude from it, and that documented meaning is what the
    API guarantees. Final means the transaction's block is final on the
    underlying L1, so a caller acting on it is acting on a settled result; it is
    not a statement about how much of the chain a node has processed. Pending
    means the transaction is in the pending set the node holds or observes
    (#35). Where LEZ recognises a degree of certainty between inclusion and
    finality, as `BedrockStatus` does with `Safe`, the level set carries it
    rather than collapsing it into either neighbour. The node today reads zone
    blocks only once they are final on L1
    (`lez/chain_state/src/zone_indexer.rs:54-86`) and stamps every block it
    stores `Finalized` (`lez/indexer/core/src/block_store.rs:345`), so the
    levels short of final are not yet observable from it. A proposal states how
    the node learns of a block before it is final: from the L1, or, in sequencer
    mode, from the blocks the node produces. **[New]**
23. The per-transaction status query returns the identifier of the block
    containing the transaction, which the indexer store already resolves from
    the transaction hash (`lez/indexer/core/src/block_store.rs:144-146`)
    **[Ready, not exposed]**, and the L1 position the level was assessed
    against, so a caller can tell how current the answer is **[New]**.
24. The per-transaction status query reports execution success or failure, with
    a typed reason on failure, for any transaction the node has executed. A
    charged transaction whose action fails is included in its block, keeps its
    fee, and is listed in the block's reverted indices
    (`lez/indexer/service/protocol/src/lib.rs:207-213`), which the indexer RPC
    serves and the FFI block type drops; that outcome is **\[Ready, not
    exposed\]**. The reason is computed as free text when the action fails and
    then discarded (`lee/state_machine/src/validated_state_diff/mod.rs:69-75`),
    so a typed reason is **[Computed, not persisted]**.
25. A transaction this node has seen, because it was submitted through this node
    or observed in its pending set, and that can no longer be included, is
    reported as dropped rather than as not known. A public transaction can no
    longer be included once the nonce of an account it consumes has moved past
    the nonce it declares (`lez/indexer/service/protocol/src/lib.rs:256`), and a
    privacy-preserving one once its block or timestamp validity window has
    passed (`lez/indexer/service/protocol/src/lib.rs:357-358`). For the
    transactions it has seen, the node keeps what it needs to tell, and the API
    documents for how long. **[New]**
26. The API documents that a transaction this node never saw, absent from the
    chain and from the pending set, is reported as not known, whether it was
    never submitted or was submitted elsewhere and dropped, no record of a
    declined transaction being retained. **[New]**

**`query_status() -> *mut c_char`**

Returns the node's own ingestion state as a JSON document: `state`,
`indexed_block_id`, `last_error`, `stall_reason`, `cross_zone_halt`, and
`cross_zone_peers`.

27. The query reports the ingestion state as one of `Starting`, `Syncing`,
    `CaughtUp`, `Error`, `Stalled`, or `Halted`, together with the indexed block
    identifier, and carries the stall and cross-zone diagnostics when ingestion
    stopped. A state a client build does not recognise is treated as not known
    healthy rather than as a decode failure. **[Ready]**
28. The status query reports the node's mode: whether it is in sequencer mode
    and, if it is, whether it is accepting transactions or still starting, as
    the sequencer service's `getSequencerStatus` reports today
    (`lez/sequencer/service/rpc/src/lib.rs:151-152`); and, outside sequencer
    mode, whether the node has a peer to relay a submission through. A caller
    learns before submitting whether its transaction is admitted here or
    relayed, and whether it can be either. **[New]**
29. The status query signals a null handle and a serialisation failure as
    distinguishable outcomes rather than returning a bare null pointer for both
    ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
    **[New]**

##### Transaction submission

Handing a signed transaction to the network
([Appendix: Blockchain API and SDK Ecosystem, section 1.21](../appendix/blockchain-api-sdk-ecosystem.md#121-broadcast-a-signed-transaction)).

- **Ethereum**: `eth_sendRawTransaction`.
- **Solana**: `sendTransaction`.
- **Bitcoin**: `sendrawtransaction`.
- **Zcash**: `sendrawtransaction`, inherited from Bitcoin.

Every surveyed chain accepts a signed transaction at the same endpoint a caller
reads from. A wallet that builds and signs a transaction has to hand it
somewhere, and since the node API is the only surface a consumer reaches, that
somewhere is the node: a caller that had to reach past it to submit would be
reaching into LEZ for the one operation the read surface does not cover.

This does not make the node a writer of chain state. It accepts a transaction
another party constructed and signed, and takes it from there. Nothing here
constructs, signs, or executes a transaction.

**`submit_transaction(transaction) -> PointerResult<FfiBytes32, OperationStatus>`**

Accepts a signed transaction and returns the hash it will be known by.

30. The FFI accepts a signed transaction and returns the transaction hash a
    caller then tracks through `query_transaction_status`. A node in sequencer
    mode admits the transaction to its own pending set; any other node relays it
    over the LEZ peer-to-peer network to the nodes that sequence, which is not
    part of this API. The node FFI has no such function. The sequencer service's
    FFI in `core_ffi` has one, `sequencer_ffi_send_transaction`
    (`lez/core_ffi/src/sequencer/api/query.rs:408-438`), which is prior art for
    the shape. **[New]**
31. A transaction the node declines is reported as a typed failure,
    distinguishing at least a malformed transaction, one that fails a stateless
    check, one too large for a block, one whose fee is below the current price
    the node knows, and one invoking a program a user transaction may not call,
    separately from a node that cannot take a submission at all: one still
    starting, or, outside sequencer mode, one with no peer to relay through. A
    node in sequencer mode also reports its own admission: a pending set that is
    full, and a privacy-preserving fee reserve below the fee at the head. The
    sequencer service's RPC gives three outcomes codes of their own today, a
    full mempool, a privacy-preserving reserve below the head fee, and not yet
    online (`lez/sequencer/service/rpc/src/lib.rs:18-27`), and reports a public
    fee shortfall, an oversized transaction, and a call to a sequencer-only
    program as invalid parameters with text
    (`lez/sequencer/actors/rpc_server/src/actor/service.rs:58-68`,
    `lez/sequencer/actors/rpc_server/src/actor/service.rs:456-464`), while the
    sequencer FFI reports every failure alike. A caller re-prices on an
    insufficient fee, retries on a full pending set or a node that cannot take a
    submission yet, and does not retry the others. **[New]**
32. Outside sequencer mode, acceptance means the node relayed the transaction,
    not that a sequencer admitted it, and a sequencer's later refusal does not
    come back through this call. The API documents this, and the caller learns
    what became of the transaction from the status query (#22, #25). **[New]**
33. Submission neither signs nor modifies the transaction it is given. What is
    relayed is the transaction accepted, re-encoded at most, and the hash
    returned is the hash of that transaction. **[New]**

##### Pending transactions

Listing what the network holds but has not yet included in a block
([Appendix: Blockchain API and SDK Ecosystem, section 1.26](../appendix/blockchain-api-sdk-ecosystem.md#126-inspect-the-mempool-or-pending-set)).

- **Bitcoin**: `getrawmempool` lists the set, `getmempoolentry` returns one
  entry's detail, `getmempoolinfo` its size.
- **Zcash**: `getrawmempool`, `getmempoolinfo`, and `getaddressmempool`, the
  last insight-explorer gated.
- **Cosmos**: `/unconfirmed_txs` and `/num_unconfirmed_txs`.
- **Ethereum**: `eth_newPendingTransactionFilter`, with no direct dump.
- **Solana, XRPL, Stellar, NEAR, Sui**: no equivalent. Solana has no mempool at
  all: transactions are forwarded to the upcoming leaders.

A privacy-preserving transaction commits to a post-state when it is built, so a
pending transaction that changes the pre-state it assumed invalidates it before
it is ever submitted. Generating the proof takes minutes, so a wallet that can
see the pending set aborts a proof it now knows will be rejected instead of
finishing it. That is the difference between reading the pending set and
diagnosing a failure afterwards.

The pending set also answers where a submitted transaction has got to. Absence
from the chain alone means nothing, since a transaction not yet included and one
the network declined look identical, and only four of the nine surveyed chains
expose a pending set to tell them apart. A charged transaction whose action
fails during block production is included as reverted, but one that fails
settlement is dropped without a record: it is logged and omitted from the block
(`lez/sequencer/core/src/lib.rs:1792-1803`), so no drop is retrievable
afterwards and the pending set is the only surface on which its disappearance is
visible. It is the proof a wallet abandons, though, rather than status, that
makes the pending set a requirement here.

Which pending set a node can show depends on its mode. A sequencer's pending set
is a bounded mempool, fed both by submission and by gossip from its peers, and
it is only pushed to today (`lez/mempool/src/lib.rs:66-92`). A node outside
sequencer mode holds none, but the transactions sequencers gossip travel on a
peer-to-peer topic per zone (`lez/sequencer/actors/gossip/src/actor.rs:336-338`)
that any node following the zone can observe. Either way, one transaction sits
in several pending sets at once, none of them observable today, until it appears
in a block
([Appendix: Blockchain API and SDK Ecosystem, section 1.26](../appendix/blockchain-api-sdk-ecosystem.md#126-inspect-the-mempool-or-pending-set)).

**`query_pending_digests(cursor, limit) -> PointerResult<FfiVec<FfiPendingDigest>, OperationStatus>`**

Lists the pending set as digests: per entry, its transaction hash and the
account identifiers, nonces, and nullifiers it touches.

34. The FFI lists pending transactions as digests carrying, per entry, its
    transaction hash together with the account identifiers, nonces, and
    nullifiers the transaction acts on, so a caller detects a conflict with a
    transaction it is about to build without fetching bodies. **[New]**
35. In sequencer mode the listing is the pending set the node builds blocks
    from. Outside it, the listing is the transactions the node has observed on
    the LEZ peer-to-peer network and not yet seen included, together with those
    submitted through it. The API documents which a node serves, and that in
    neither mode does absence from the listing show that a transaction is not
    pending elsewhere. **[New]**
36. The listing is bounded and paginated by cursor, on the same terms as the
    other paginated reads, and reports whether more entries remain. The pending
    set turns over as entries are included or dropped, so an offset into it
    names a different entry from one call to the next. **[New]**
37. A page of the digest listing costs no more to serve than a page of blocks of
    the same limit, measured under Performance #1, so a wallet re-checks for a
    conflict throughout a proof generation and abandons the proof rather than
    completing one it knows will be rejected. **[New]**
38. The pending set is queryable by transaction hash, so a caller holding the
    hash `submit_transaction` returned asks whether that transaction is still
    pending without walking the listing. **[New]**

**`query_pending_entry(hash) -> PointerResult<FfiOption<FfiTransaction>, OperationStatus>`**

Returns one pending transaction in full, by its transaction hash.

39. The FFI returns a pending transaction's body for a transaction hash, in the
    same shape `query_transaction` returns an included one, and reports absence
    distinctly from failure: an entry included or dropped since the caller last
    saw it is gone rather than an error. **[New]**

##### Subscriptions

Pushing new blocks and events to a consumer instead of making it poll
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

40. The FFI exposes a subscription to finalised blocks that delivers each newly
    indexed block to a registered consumer, exporting the existing
    `subscribeToFinalizedBlocks` indexer RPC method. The consumer is notified
    through a callback rather than by polling. **[Ready, not exposed]**
41. Each delivery carries the block's height and its hash together, so a
    consumer can check the block against one it already holds without a second
    read. The existing subscription yields a height alone
    (`lez/indexer/service/rpc/src/lib.rs:44`); the pair already exists as
    `BlockMeta` (`lez/common/src/block.rs:11-14`). **[New]**
42. The block subscription accepts a start position: the identifier and hash of
    the block the consumer last processed. Delivery resumes from the block after
    that position, so a consumer that reconnects observes no gap, and a position
    whose hash does not match the block the node holds at that identifier is
    reported as such rather than resumed from. Neither existing RPC subscription
    carries a position: `subscribeToFinalizedBlocks` takes no arguments, and
    `subscribeToEvents` filters by transaction, program, and selector but not by
    position
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.33](../appendix/blockchain-api-sdk-ecosystem.md#133-resume-a-stream-from-a-known-position)).
    **[New]**

**`unsubscribe(subscription) -> OperationStatus`**

Cancels a registered subscription and releases the resources it holds.

43. Cancelling a subscription stops delivery, and no callback fires after the
    call returns. **[New]**

**`subscribe_to_events(filter, from_block, callback, user_data) -> PointerResult<FfiSubscription, OperationStatus>`**

Registers a consumer that is called with each new event record matching a
filter, optionally resuming from a position the consumer already processed.

44. The FFI exposes a subscription to program events, exporting the existing
    `subscribeToEvents` indexer RPC method
    (`lez/indexer/service/rpc/src/lib.rs:47-48`) with the filter it takes by
    transaction, program, and selector, and delivering each record in the shape
    `query_events` returns. Cancellation, and delivery to the caller that opened
    the subscription alone, are on the same terms as the block subscription.
    Today the subscription stops at the indexer RPC, so no application can
    follow a program's events. **[Ready, not exposed]**
45. The event subscription resumes from a block identifier and hash on the same
    terms as the block subscription, #42, so a consumer following a program's
    events across a disconnection misses none. **[New]**

##### Pagination and account history

Walking an account's transaction history in bounded pages
([Appendix: Blockchain API and SDK Ecosystem, section 1.30](../appendix/blockchain-api-sdk-ecosystem.md#130-paginate-a-result-set)
and
[section 1.31](../appendix/blockchain-api-sdk-ecosystem.md#131-query-historical-transactions-for-an-address)).

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

46. The query returns at most `limit` transactions from the position the cursor
    names, or from the start of the walk when the cursor is absent, and stops
    early at the end of the account's history rather than failing. The bounded
    read exists; the cursor replaces the offset it takes today. **[New]**
47. `query_transactions_by_account` accepts an ordering parameter supporting
    both oldest-first and newest-first. Newest-first is the order a deposit
    tracker reads in. **[New]**
48. The page is walked by an opaque cursor rather than by a numeric offset, and
    the cursor remains stable across ingestion: a caller resuming from one
    neither skips nor repeats an entry that existed when the walk began, however
    many transactions have landed since. An offset into a growing index cannot
    hold that property, because entries arriving ahead of the offset shift every
    later position, and a deposit tracker reading newest-first is the case that
    breaks first. The cursor is opaque to the caller, which may not construct
    one or infer a position from it. **[New]**

**`query_blocks(from, limit, order) -> PointerResult<FfiVec<FfiBlock>, OperationStatus>`**

Returns a page of blocks after `from`, `from` itself excluded, walked in the
requested direction. With `from` absent, a newest-first walk starts at the
indexed tip and an oldest-first walk at the retention floor. Exported today as
`query_block_vec`, which descends only.

49. The query returns at most `limit` blocks, descending from `from`, or from
    the indexed tip when `from` is absent. **[Ready]**
50. The bound is documented as exclusive in both directions. The store already
    implements it that way when descending, from `before_id.saturating_sub(1)`
    (`lez/storage/src/indexer/read_multiple.rs:11`), leaving the documentation
    obligation only. **[Ready]**
51. `query_blocks` accepts an ordering parameter supporting both oldest-first
    and newest-first, on the same terms as `query_transactions_by_account`.
    Oldest-first is the order a consumer scanning forward reads in: a wallet
    syncing private accounts walks ascending from the last block it processed to
    the tip, decrypting each privacy-preserving transaction body against its own
    viewing key. Descending from the tip cannot serve that walk. An ascending
    walk with `from` absent starts at the retention floor. **[New]**
52. Every paginated response reports whether more results remain, and carries
    the position a caller resumes from, so a caller distinguishes the end of a
    result set from a page that happens to be short. For the account history and
    the pending listing that position is an opaque cursor the caller does not
    construct (#48); for the walks keyed by position, the blocks, publications,
    and encrypted notes, it is the block or commitment-set position the next
    page starts after. No paginated return type carries either signal today.
    **[New]**

##### Chain and node metadata

What an integrator needs to know about the node it is talking to: where the
chain ends, which network this is, what the fee market currently charges, and
how far back the data goes
([Appendix: Blockchain API and SDK Ecosystem, section 1.3](../appendix/blockchain-api-sdk-ecosystem.md#13-identify-the-network-or-chain)
and
[section 1.6](../appendix/blockchain-api-sdk-ecosystem.md#16-get-the-current-chain-tip)).

- **Ethereum**: `eth_blockNumber` and `eth_chainId`.
- **Solana**: `getSlot` and `getGenesisHash`.
- **Bitcoin**: `getblockcount` and `getblockchaininfo`.
- **Zcash**: `getbestblockhash` and `getblockcount` for the tip,
  `getblockchaininfo` for the network.

**`query_version() -> PointerResult<FfiVersion, OperationStatus>`**

Returns the software the indexer is running, so a caller knows which build
answered it
([Appendix: Blockchain API and SDK Ecosystem, section 1.1](../appendix/blockchain-api-sdk-ecosystem.md#11-get-node-version-and-software-identity)).
Every surveyed chain reports this and LEZ reports none: `web3_clientVersion` on
Ethereum, `getVersion` on Solana, `getnetworkinfo` on Bitcoin and Zcash.

53. The FFI reports the version of the node serving the call, together with
    enough build identity to tell two builds of the same version apart. An
    integrator that hits a defect can then say which build produced it, and one
    that knows a build is bad can route around it. Neither the FFI's
    `query_status` nor the indexer RPC's `checkHealth`, which the FFI does not
    export, reports a version
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**
54. The version changes only across restarts, so a caller may cache it for the
    life of a connection and use it to detect that the surface it holds a schema
    for has been replaced. **[New]**

**`query_block(block_id) -> PointerResult<FfiBlockOpt, OperationStatus>`**

Returns the block at a given identifier: its header, its full transaction body,
and its bedrock status.

55. The query returns the stored block for a known identifier and an absent
    `FfiBlockOpt` for one the indexer does not hold, keeping the two distinct
    from a backend failure. **[Ready]**
56. The returned header carries the block identifier, previous block hash, own
    hash, and timestamp (`lez/indexer/service/protocol/src/lib.rs:178-184`), and
    the body carries every transaction in the block. **[Ready]**
57. The returned body carries the indices of the transactions whose action
    reverted, which the stored block and the indexer RPC already carry
    (`lez/indexer/service/protocol/src/lib.rs:207-213`) and the FFI block type
    drops (`lez/indexer/ffi/src/api/types/block.rs:15-33`). Without them a
    consumer reading a block cannot tell a transaction that took effect from one
    that only paid its fee. **[Ready, not exposed]**

**`query_block_by_hash(hash) -> PointerResult<FfiBlockOpt, OperationStatus>`**

Returns the same block record, resolved by block hash rather than by height. The
store already holds the hash as a secondary index onto the height
(`lez/storage/src/indexer/read_once.rs:67-70`).

58. The query returns the same record `query_block` returns for the
    corresponding height, and an absent `FfiBlockOpt` for a hash no indexed
    block carries. **[Ready]**

**`query_last_block() -> LastBlockIdResult`**

Returns the identifier of the node's last finalised block, inline, with no
allocation to free.

59. The query returns the last finalised block identifier, and reports an empty
    chain as an outcome distinct from an error. **[Ready]**

**`query_chain_tip() -> PointerResult<FfiChainTip, OperationStatus>`**

Returns the tip as one record rather than as a height the caller must then
resolve.

60. The FFI exposes the chain tip as a single call returning the tip block's
    height together with its hash and timestamp, so learning about the tip does
    not cost a second call and a caller knows which block holds the position. No
    such record exists below the FFI: `getLastFinalizedBlockId` returns a height
    alone, and the header fields come from a second read. **[New]**

**`query_network_identity() -> PointerResult<FfiNetworkIdentity, OperationStatus>`**

Returns what the node is following: which zone, and which Logos Blockchain chain
that zone settles to.

61. The call returns the zone identifier the node is following, so an
    application can confirm which zone it is connected to. The node's channel
    configuration already holds it (`lez/indexer/core/src/config.rs:37`).
    **[Ready, not exposed]**
62. The same call returns the chain identifier of the Logos Blockchain the zone
    settles to. A zone identifier alone does not distinguish the same zone
    running against different L1 networks, which is the case an integrator
    connecting to the wrong network hits first, and the two are answered
    together so a caller cannot check one and assume the other. **[New]**
63. The chain identifier is the one inscribed in the L1 genesis block as a
    Cryptarchia parameter, a bounded UTF-8 string such as `logos-chain-1`, and
    not a value the indexer is configured with independently: a configured
    string would agree with whatever an operator typed rather than with the
    chain the node is settling to. It is inscribed and read at ledger
    initialisation but served by no L1 route
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.3](../appendix/blockchain-api-sdk-ecosystem.md#13-identify-the-network-or-chain)),
    and the indexer holds only an endpoint, with optional credentials, for its
    Bedrock connection (`lez/indexer/core/src/config.rs:25-30`), so obtaining it
    is work outside the indexer. A proposal states how it reaches the value.
    **[New]**
64. Where the chain identifier cannot be obtained, the call reports it as
    unavailable rather than omitting it or returning a placeholder, so a caller
    can tell an unidentified chain from an unasked question. **[New]**

**`query_program_ids() -> PointerResult<FfiVec<FfiProgramEntry>, OperationStatus>`**

Returns the programs deployed on the chain, each with its account identifier,
its image identifier, and, for a builtin, its name, together with the identifier
of the privacy-preserving circuit the chain verifies proofs against.

65. The FFI exposes the programs deployed on the chain as its state records
    them, programs deployed after genesis included. A program is an account
    written through the program loader, and the state resolves an account to the
    image deployed at it (`lee/state_machine/src/state/mod.rs:400-420`). An
    integrator decoding account data needs this: each actor state is keyed by
    the account identifier of its program, and decoding it means knowing which
    program that is. The sequencer service's `getProgramIds` returns a fixed
    list today (`lez/sequencer/actors/rpc_server/src/actor/service.rs:264-272`).
    **[New]**
66. The same call returns the identifier of the privacy-preserving circuit whose
    proofs the chain accepts, which a wallet compares with its own before it
    proves: a mismatch otherwise surfaces only when a transaction that took
    minutes to prove is rejected. That identifier is the one entry
    `getProgramIds` returns today, and the wallet's health check compares it
    with its own (`lez/wallet/src/cli/mod.rs:299-308`). **[Ready, not exposed]**

**`query_fee_state() -> PointerResult<FfiFeeState, OperationStatus>`**

Returns the fee market a transaction is priced against: the base fees at a
stated block, the band the next block's may move within, and the per-block gas
caps
([Appendix: Blockchain API and SDK Ecosystem, section 1.17](../appendix/blockchain-api-sdk-ecosystem.md#117-estimate-the-fee-or-fee-rate)).

67. The FFI exposes the execution and storage base fees, the block height they
    were read at, and the per-block execution and storage gas caps, from the fee
    state the chain state holds (`lez/chain_state/src/apply.rs:332-341`). The
    wallet prices every fee declaration and every in-proof reserve against these
    values, read today from the sequencer service's `getFeeState`
    (`lez/sequencer/service/rpc/src/lib.rs:57-58`), and the node FFI serves none
    of them. **[Ready, not exposed]**
68. The same call returns the lowest and highest base fees the next block can
    open at, in each dimension, as `getFeeState` does, so a caller sets a cap on
    the fee from the band rather than from a guess. The band follows from the
    fee state and the fee market's constants
    (`lez/sequencer/core/src/fees.rs:89-123`). The API documents that the band
    bounds the next block and is not an estimate for a given transaction.
    **[Ready, not exposed]**

No tip history is required. A sequencer takes transactions from its pending set
in arrival order (`lez/mempool/src/lib.rs:40-58`,
`lez/sequencer/core/src/lib.rs:1590`), so at the commit read the tip does not
buy earlier inclusion, and the API documents what it does buy. Soft #6 covers
the case where inclusion comes to depend on it.

**`query_retention_floor() -> PointerResult<FfiRetentionFloor, OperationStatus>`**

Returns the earliest block the node can still answer for, separately for account
state pinned to a block and for transaction retrieval.

69. The FFI exposes the node's retention floor: the earliest block for which
    account state can be read at a pinned block identifier, and the earliest
    block for which a transaction can be retrieved. A deployment that retains
    everything reports genesis and one bounded by Reliability #7 reports a
    moving value, and a caller learns which by asking rather than by discovering
    it through a failed read. Where a deployment keeps some data further back
    than the rest, as Soft #5 allows, the call reports retention per scope
    rather than one floor that is wrong for part of the data. **[New]**
70. A read for data below the retention floor is answered as such, distinctly
    from a read for data that never existed. Asking for a block the indexer
    discarded and asking for one the chain never held are different questions
    with different answers: the first is retried against an archival source and
    the second is not. A caller learns which without a second call to the
    retention floor, which may itself have moved between the two. **[New]**

##### Private message delivery

Reading the messages a private account receives. A send to a private account is
published rather than applied: a public-to-private send as a clear publication
whose envelope is in plaintext, and a private-to-private send as a sealed one,
each at a position in the commitment set. The receiving wallet reads the
publications, finds those addressed to it, and later consumes them. Among the
surveyed chains only Zcash, whose shielded payments a recipient finds by
scanning, has a comparable structure. The sequencer service serves it today
(`lez/sequencer/service/rpc/src/lib.rs:105-122`), and the indexer holds the data
in its own state (`lee/state_machine/src/state/mod.rs:363-398`).

**`query_publications(from_position, limit) -> PointerResult<FfiPublicationPage, OperationStatus>`**

Returns a page of publications in commitment-set order, from a given position.

71. The FFI lists publications in position order from a given position, each
    with its position: for a clear publication its plaintext envelope and its
    recovery note, and for a sealed one its commitment and encrypted note.
    **[Ready, not exposed]**
72. The listing is bounded and reports whether more entries remain, on the same
    terms as the other paginated reads, and takes no account identifier, so a
    wallet receiving messages discloses nothing beyond the position it resumes
    from. **[New]**

**`query_recovery_binding(address) -> PointerResult<FfiOption<FfiEncryptedNote>, OperationStatus>`**

Returns the encrypted note bound to a private address, if one is.

73. The FFI returns the recovery note bound to a private address, or an absent
    `FfiOption` when none is, exporting what the sequencer's
    `getRecoveryBinding` serves today. **[Ready, not exposed]**
74. The API documents that this lookup names a private address to the node, and
    that a consumer which must not disclose its addresses reads the same notes
    from the publication listing or the viewing key walk instead, neither of
    which takes an address. **[New]**

##### Viewing key reads

Reading what a viewing key can open, without the node holding the key, for a
consumer that holds a viewing key and no spending key (see
[Reading with a viewing key alone](#reading-with-a-viewing-key-alone)). The
publication listing above covers received messages; this walk covers every
encrypted note the chain carries, account state updates included.

- **Zcash**: `z_importviewingkey` imports the key into the node's wallet, which
  then answers for it; lightwalletd instead streams compact blocks a light
  client scans on its own side ([ZIP 307](https://zips.z.cash/zip-0307)).
- **Ethereum, Solana, Bitcoin**: no equivalent, there being no encrypted state
  to scan.

**`query_private_notes(from_block, limit) -> PointerResult<FfiNotePage, OperationStatus>`**

Returns, block by block in ascending order, every encrypted note the chain
carries with the public values needed to decrypt and check it.

75. The FFI exposes a walk, ascending by block from a given block, returning for
    each privacy-preserving transaction its hash, its index in the block, and
    whether its action reverted, and for each encrypted note it carries the
    values a viewing key holder needs to open and check it: for a private
    action, including the fee fallback's, its nullifier, commitment, and
    encrypted post-state; for a sealed async message, its commitment, note, and
    position in the commitment set; and for a recovery binding, its address and
    note. Proofs, witness sets, and public bodies are omitted; a clear
    publication, whose envelope is plaintext, is read from the publication
    listing, #71. Every field but the position is already in the stored
    transaction (`lez/indexer/service/protocol/src/lib.rs:323-395`), and the
    position is in the indexer's publication state, so the work is a compact
    projection of data the indexer holds. **[New]**
76. The walk covers every encrypted note whatever kind of private account it
    addresses: a regular private account, a private PDA, and a shared private
    account keyed from a group master secret, of either kind. The node needs no
    knowledge of which kind a note belongs to, and the walk does not depend on
    it. **[New]**
77. The walk takes no viewing key, account identifier, or other value tied to
    the consumer's accounts. Every caller asking for the same range receives the
    same result, so the walk discloses to the node the range read and nothing
    else. **[New]**
78. The walk resumes from a block identifier and hash on the same terms as the
    block subscription, #42, and is also delivered as a subscription, so a
    viewing key consumer following the tip neither polls nor misses a block.
    **[New]**
79. The API documents what a consumer holding a viewing key alone can establish
    from the walk and what it cannot. It can establish every state of the
    account that a note encrypted to its viewing key carries, and every message
    sealed to it. The circuit encrypts each private action's post-state under
    the viewing key its witness holds
    (`lee/privacy_preserving_circuit/src/output.rs:197-245`), and a regular
    private account's identifier commits to that key, so every update of a
    regular private account produces a note its viewing key opens; the
    documentation states whether the same holds for a private PDA and for a
    shared account. It cannot establish whether a note it opened has since been
    consumed, other than by seeing the state that replaced it, because every
    nullifier is keyed by a secret the viewing key does not include
    (`lee/state_machine/core/src/nullifier.rs:126-158`). **[New]**

##### Surface-wide obligations

These bind every function above rather than adding one
([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).

- **Ethereum**: JSON-RPC error codes, and OpenRPC as the published contract.
- **Solana**: JSON-RPC 2.0 error objects with numeric codes.
- **Bitcoin**: numeric JSON-RPC error codes, with no published machine-readable
  contract.
- **Zcash**: the Bitcoin-inherited numeric error codes, with no Zcash-specific
  taxonomy published.

The module parity requirement has no ecosystem analogue, because no surveyed
chain interposes a plugin layer between its API and its consumers.

**The module contract**

What a caller generates its client from, and what Logos Core peering checks both
sides agree on.

80. `lez_indexer_module` publishes its contract as a CDDL schema, as every Logos
    module does, covering every method and event the module exposes: its
    parameters, its return type, its error values, the set it belongs to (#89),
    and the node modes it is available in (#83). The module describes the
    contract at run time, and each release documents which FFI function serves
    each module method. The existing `getSchema` describes the block, event
    record, and event filter types rather than the surface, and is not exported
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**

Every function defined above is further bound by the following.

81. Every FFI function that takes the indexer handle returned by
    `start_indexer`, as its first argument the signatures above elide, signals a
    null handle as `OperationStatus::NullPointer` rather than as a not-found
    result. Every query exported today does so except `query_status`, which #29
    addresses. **[Ready]**
82. Every read reports the indexed tip it was served against: the identifier of
    the last block the node had indexed when it answered. A caller detects a
    stalled or lagging node from any response, not only from the status query.
    No return type carries it today. **[New]**
83. Every function is available on every node unless the contract (#80) marks
    it, or one of its outcomes, for sequencer mode. A node outside sequencer
    mode answers such a call as not available in its mode, an outcome distinct
    from not found, invalid argument, backend failure, and refusal (#85, #93),
    so a caller learns to call a node in sequencer mode rather than retrying
    this one. **[New]**
84. Every function above is exposed through `lez_indexer_module` with the same
    semantics, including the not-found and error distinction required by
    Functionality #85. No capability reaching the FFI stops at the module
    boundary. Today the module wraps eight of the FFI's nine queries, omitting
    `query_events`, and is built against the LEZ `v0.3.1` FFI rather than the
    testnet 0.4 one. **[New]**
85. The FFI and the module signal not-found, invalid-argument, and backend
    failure as three distinguishable outcomes on every query. The module
    currently flattens not-found and failure into an empty string
    ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
    **[New]**
86. Errors are machine-readable and stable: a caller branches on a value rather
    than on message text, and that value does not change meaning between
    releases. Which values exist is for the implementation to decide; what this
    RFP requires is that two failures a caller would recover from differently
    never arrive as the same value. The indexer RPC returns the stock JSON-RPC
    `InternalError` code with free text for most failures, typing only an
    invalid event range, and the FFI's status type has five values, so most
    failures look alike
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).
    **[New]**
87. The distinctions a caller acts on are, at minimum: a request that was
    malformed, which the caller fixes; a request for data below the retention
    floor, which the caller sends elsewhere; a request the indexer could not
    serve because it is stalled or lagging, which the caller retries; and a
    backend failure, which the caller retries with backoff. A transient failure
    is distinguishable from a permanent one without parsing text. **[New]**
88. Every heap-allocating return has a documented matching free function, and
    calling it releases every allocation the return holds. The returns exported
    today have theirs **[Ready]**; the return types this RFP adds need theirs
    **[New]**.

##### Access control and concurrent callers

Who may call which function, and what holds when several wallets call the same
node.

- **Ethereum**: Geth groups methods into namespaces and exposes them per
  transport. Local IPC serves every namespace, and HTTP serves `eth`, `net`, and
  `web3` unless `--http.api` names others, so `admin`, `debug`, and `txpool`
  stay off a default HTTP endpoint.
- **Solana**: the validator's administrative calls, `exit` among them, are
  served over a local IPC socket in the ledger directory and not on the public
  JSON-RPC port.
- **Bitcoin**: `stop` sits on the same RPC as the reads. `-rpcauth` defines
  several users and `-rpcwhitelist` restricts each to a list of methods, which
  is the closest precedent to a per method grant.
- **Zcash**: `zcashd` inherits `stop` from Bitcoin on the same RPC. Many wallets
  reach one node through lightwalletd, which exposes reads and transaction
  submission and no administrative call.

**`start_indexer(runtime, config_path, storage_dir) -> PointerResult<IndexerServiceFFI, OperationStatus>`**

Starts the node against a configuration and a storage directory and returns the
handle every other function takes. It is the one function that takes no handle.

**`stop_indexer() -> OperationStatus`**

Stops the node and releases the handle.

**`reset_storage(config_path)`**

Stops the node and deletes the store of the zone the configuration names,
without restarting it. Exposed by `lez_indexer_module` only.

89. Every function this RFP defines belongs to exactly one of two sets, and the
    contract of #80 states which. The user set holds every function defined
    under the headings above. The administrative set holds `start_indexer`,
    `stop_indexer`, and `reset_storage`, together with any other function the
    module exposes that changes how the node runs, such as the FFI's
    `init_logger` were the module to expose it. A function added to the surface
    later is assigned to a set when it is added. **[New]**
90. Each function exposed through `lez_indexer_module` is individually
    grantable. Where the deployer's Logos Core access policy grants a caller a
    list of methods on the module, that caller reaches those methods and no
    others, and the module is built so that such a grant is enforced against it
    rather than refused as unenforceable. **[New]**
91. An administrative function refuses a call unless the access policy grants
    the caller that function by name, or the caller is the runtime, its shell,
    or an operator of that runtime. The refusal holds when no access policy is
    in force, when the policy leaves the module open to every caller, and when
    the runtime cannot identify the caller. A caller on another runtime is never
    the runtime, its shell, or its operator, so it passes only on a grant of the
    function by name. **[New]**
92. A user function requires nothing of a caller beyond admission to the module.
    No user function requires a caller to register, to name a wallet, or to have
    made an earlier call, and the node holds no state for a caller other than
    the subscriptions that caller opened. A cursor carries the position it
    resumes from, so a walk part way through holds nothing on the node.
    **[New]**
93. A refused call is reported as an outcome distinct from the not-found,
    invalid-argument, and backend failure outcomes of #85, and has no effect on
    the node. A caller does not retry it. **[New]**
94. Calls from different callers are served concurrently. No call waits on
    another caller's call, page walk, or subscription callback, and a cursor
    issued to one caller is neither advanced nor invalidated by another caller's
    calls. **[New]**
95. A subscription belongs to the caller that opened it. `unsubscribe` cancels
    only a subscription the caller itself holds, and one caller's cancellation,
    disconnection, or failing callback neither stops nor delays delivery to any
    other caller. **[New]**
96. Once the node is stopped, every call reaching the module from every caller
    reports the node as stopped, distinctly from a backend failure, and each
    open subscription receives a final notification that delivery has ended. The
    FFI handle is released on stop, so this is the module's obligation rather
    than the FFI's. **[New]**

#### Usability

1. Provide a Logos module surface (`lez_indexer_module`) covering every function
   in Functionality, usable to build Logos modules for reading LEZ state,
   following the chain, and tracking deposits.
2. Provide a CLI that covers core functionality: read an account at current and
   at a pinned block, read a batch of accounts, read a block, read a
   transaction, read a transaction's effects, query a transaction's status, list
   an account's transactions with ordering and pagination, follow the chain tip,
   follow a program's events, read the node's status and mode, read the fee
   market, list the pending set, list publications, and walk the encrypted notes
   of a block range. The CLI may have fewer features than the FFI but must
   support all essential operations.
3. Provide a worked deposit-tracking example, in the CLI or as a documented
   reference consumer, that follows the chain from a chosen start block, detects
   credits to a supplied set of public accounts, and reports each credit with
   its transaction hash, amount, block, and status level.
4. Document which state is readable and which is not for privacy-preserving
   transactions, including that a public-to-private send reveals its amount and
   receiving private address but not who controls that address, and that what a
   private account holds is not trackable from indexer data without its viewing
   key.
5. Return clear, actionable error messages for every failure mode, each carrying
   the machine-readable value required by Functionality #86 alongside the text.
6. Document the semantics of every pagination parameter, including the
   exclusivity of the block bound, what a cursor guarantees across ingestion,
   and the behaviour when new data lands during a walk.
7. Provide a reference Logos Core access policy for a node serving wallets on
   its own runtime and for a node serving wallets on other runtimes. Each grants
   the user set to wallets, and the administrative set to no caller beyond those
   Functionality #91 admits.
8. Document how a deployer grants a caller a single function or a subset of the
   user set, such as the reads without `submit_transaction`.
9. Provide a worked viewing key example, as a documented reference consumer
   outside the CLI and the module, that is given a viewing key and the public
   keys of an account and nothing else, follows the chain through the viewing
   key walk and the publication listing, and reports the account's successive
   states and the messages sent to it. Decryption is the wallet's, so the
   example reuses the LEZ wallet's decryption rather than adding any to this
   delivery; what it demonstrates is that the node serves everything a viewing
   key consumer needs. The example is run once for a regular private account and
   once for a shared private account whose viewing key was derived from a group
   master secret.

#### Reliability

1. A pinned account read at a given block identifier returns the same value on
   every call, however far ingestion has advanced, and returns the hash of the
   block it answered against so a caller can tell that a repeated read resolved
   to the same block.
2. A batch account read pinned to a block identifier returns a set of accounts
   consistent with one another as at that block, not a mixture of values read at
   different points during concurrent ingestion.
3. Pagination remains valid across ingestion: a caller walking an account's
   transactions by cursor, or blocks, publications, and encrypted notes by
   position, neither skips nor repeats an entry that existed when the walk
   began.
4. A subscription consumer that disconnects and reconnects with its last
   processed block identifier and hash receives every block after that position,
   with no gap and no assumption that the consumer was connected.
5. A stalled or lagging node is detectable from any read, through the indexed
   tip every response carries (Functionality #82).
6. A stalled indexer is reported as stalled by the status surface rather than
   serving stale reads silently.
7. Storage growth is bounded by something the deployment controls rather than by
   the length of the chain. The indexer runs both as infrastructure an operator
   provisions and inside Basecamp on an end user's machine, where no one is
   watching a disk and no one can add one, so growth that a server operator
   would size for is a defect on a desktop. A deployment can put a ceiling on
   what the indexer retains, and reaching that ceiling degrades what the
   retention floor reports rather than failing reads or exhausting the disk.
8. No caller degrades the node for the others through the user set. Every user
   function bounds what one call consumes, including the size of a batch, the
   limit of a page, the span of a range, the size of a response, and the number
   of subscriptions one caller holds open. A call past a bound returns
   `InvalidArgument`, and the bounds are documented. A whole-account read
   (Functionality #10, #17) of an account whose actor states exceed the response
   bound reports the account as too large to read whole, and the caller reads it
   one program at a time (Functionality #12).

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
6. Document the latency of the user set under concurrent callers, measured
   against a stated number of callers and open subscriptions, alongside the
   single caller figures.
7. Document the size of the viewing key walk against the full blocks it is drawn
   from, in bytes per block and per encrypted note, and the time a viewing key
   consumer takes to scan a stated number of blocks, so a consumer restoring
   from a viewing key knows what a scan from genesis costs.

#### Supportability

This RFP defines the FFI. `lez_indexer_module` is what exposes it to Logos Core:
the module links `libindexer_ffi` and includes the generated `indexer_ffi.h`, so
the two ship together and a function reaches an application only when both carry
it. "The FFI and module" below means the Rust crate at `lez/indexer/ffi/` and
the module that links it.

01. The FFI and module are delivered for LEZ testnet 0.4: both are built and
    tested against a LEZ devnet and against LEZ testnet 0.4.
02. End-to-end integration tests run in CI against a local stack of a Logos
    Blockchain node and LEZ nodes, at least one in sequencer mode and one
    outside it, as the repository's `docker-compose.yml` assembles a dev Bedrock
    node, a sequencer committee, and the indexer today. Every function and
    outcome whose behaviour differs by mode is tested in each mode, including a
    call that is not available in a node's mode (Functionality #83).
03. CI must be green on the default branch.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
05. Tests cover, at minimum: a read of an identifier the state holds no record
    for; a pinned read at a block before and after a state change; a batch read
    spanning recorded and unrecorded accounts; effects for a public transaction,
    for the public leg of a privacy-preserving transaction, and for a
    transaction whose action reverted; membership proofs for a commitment and a
    message position the set holds and ones it does not; a clear and a sealed
    publication in the listing; a viewing key scan, holding no other key, of a
    regular private account recovering every state the account took, and of a
    private PDA and of a shared private account recovering what the
    documentation of Functionality #79 says a viewing key establishes for them;
    a transaction status at each documented level, a dropped public and a
    dropped privacy-preserving transaction among them; a submission to a node in
    sequencer mode and to one outside it; a subscription resumed from a stored
    position across a disconnection, and one resumed from a position whose hash
    does not match; an administrative call from a caller holding no grant, with
    no access policy in force and with a policy that leaves the module open; and
    two callers holding subscriptions when one of them cancels.
06. A README documents end-to-end usage: building the FFI and the module that
    links it, the configuration and storage directory `start_indexer` requires,
    and step-by-step instructions for every operation via the CLI.
07. Provide documentation in repo for the FFI and module, covering the developer
    integration journey for reading state, tracking deposits, and confirming
    transactions.
08. Provide documentation in repo for the CLI, covering the core operator
    journey.
09. Provide a get started guide that takes a developer from an empty repository
    to a working Logos Core module calling `lez_indexer_module`, covering the
    client generated from the module's contract, reading LEZ state through it,
    following the chain, and running the result against a devnet, with the node
    in the same runtime and reached through peering. A module calls another
    through its contract rather than linking its library, so the guide does not
    link `libindexer_ffi`. The README required above documents building and
    operating what this RFP specifies; this documents building something else on
    top of it.
10. Provide a full API reference for the indexer API, published in repo per
    version in the shape the Logos Storage module's reference takes at
    [docs.logos.co](https://docs.logos.co), covering every exported function,
    its parameters, its return type, its error codes, and the node modes it is
    available in. The reference is generated from the module contract required
    by Functionality #80 rather than maintained by hand, so it cannot drift from
    the surface it describes.
11. Provide a design decision log in repo recording each major design decision
    and choice: the context, the options considered, the option chosen, and the
    rationale and trade-offs. The log is updated as decisions are made and is
    current at each milestone delivery.

#### + Privacy

1. No function added by this RFP returns plaintext private state or a viewing
   key, or any value from which either can be recovered without the viewing key,
   and no function accepts a viewing key, a nullifier key, or a decrypted note.
   Private post-states remain readable only by the viewing key holder, who
   decrypts them on its own side.
2. The effects and status surfaces do not report, for a privacy-preserving
   transaction, any linkage between its private inputs and its private outputs
   beyond the commitments and nullifiers already published on-chain.
3. Provide a privacy properties document covering: what a third party reading
   the indexer can learn about a public transaction, about the public leg of a
   privacy-preserving transaction, including the address its fee was paid from
   under either way of paying it, and about its private leg; what a
   public-to-private send reveals; which of those boundaries are enforced by
   encryption and which by the API declining to expose data it holds; what an
   operator running their own indexer can see that a remote caller cannot; and
   what a viewing key holder can and cannot learn, for a sole-owned and for a
   shared private account.
4. No function discloses one caller's activity to another. Which accounts,
   commitments, and transactions a caller asked about, which subscriptions it
   holds and with what filter, and which caller submitted a transaction are not
   readable by any other caller through this surface.
5. The node keeps no record tying a caller identity to what that caller read or
   submitted beyond what serving the call requires. Any logging that does so is
   off by default.
6. The privacy properties document states what the operator of a node shared by
   several wallets learns about each of them from its calls: the accounts and
   commitments it reads, the transactions it submits, and the caller identity
   those arrive under.
7. A viewing key consumer can follow its accounts through calls that name none
   of them. The viewing key walk, the publication listing, and the subscription
   delivering the walk take positions only, and the documentation identifies
   every user function that takes an account identifier, a commitment, or a
   private address, so a consumer knows which calls disclose what it holds.

### Soft Requirements

If possible.

#### Functionality

1. A subscription filtered to a supplied set of accounts, delivering only blocks
   containing a transaction that touches one of them. Most surveyed chains that
   offer a block subscription also offer a filtered one; Bitcoin and Zcash push
   unfiltered streams only
   ([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).
2. A batch transaction read, taking a list of transaction hashes and returning a
   result per hash in request order.
3. A block-scoped effects query returning the diffs for every transaction in one
   block, so a consumer following the chain reads one result per block rather
   than one per transaction.
4. A decoder for the actor states returned by the account reads, so a caller can
   interpret a token balance without a client-side Borsh decode against an
   unpublished schema. `getAccount` returns each actor state as opaque bytes,
   with no bound on their size, and a token balance held through a token program
   sits in that program's actor state rather than in the native balance.
5. Retention policies beyond a single ceiling, such as keeping transaction
   history for accounts a deployment cares about while discarding the rest, so a
   desktop deployment holds what its user needs rather than the most recent
   window of everything.
6. The tips recent transactions paid, per block, so a caller chooses a tip
   against what was paid, should inclusion come to depend on the tip. The fee
   each transaction paid, which Functionality #5 persists, is what this is read
   from.
7. A read of the commitment set that names no commitment: its frontier at a
   given block, with each commitment the viewing key walk (Functionality #75)
   lists carrying its position, so a wallet keeps its own membership witnesses
   up to date and spends without naming to the node the commitments it holds, as
   a Zcash light client does from the tree state. Functionality #19 names them
   (Functionality #21).

### Out of Scope

The following are explicitly excluded from this RFP:

- **Wallet and key management.** Key handling, derivation, watch-only address
  derivation, viewing keys, and signing belong to the LEZ wallet API, a separate
  component outside this RFP. That includes how a viewing key is exported and
  imported, a view-only mode for the wallet, which LEZ does not have today, and
  whether a group of controllers should hold a viewing secret of its own rather
  than a per-account viewing key derived from its master secret. This RFP
  requires only that the node serve what a viewing key consumer decrypts.
- **Transaction construction and signing.** Building a transaction and signing
  it run through the wallet path (`core_ffi` and `lez_core`). This RFP requires
  the node to accept a signed transaction and admit or relay it, and nothing
  more: it does not construct, sign, or execute one. Verifying a signature, and
  encoding or decoding transaction bytes, which several surveyed chains offer on
  the node, belong with construction in the LEZ wallet API.
- **Communication between LEZ nodes.** Relaying a submitted transaction to the
  nodes that sequence, gossiping transactions, propagating blocks, and
  coordinating the sequencers of a zone run over the LEZ peer-to-peer network
  and are not part of this API. This API is what a wallet or application calls
  on one node. Platform Dependencies records what it relies on from that
  network.
- **Operating sequencer mode.** The reads and actions specific to running a
  sequencer, such as its committee, approvals, slashing, and cross-zone dead
  letters, are not part of this API. A node in sequencer mode serves the API
  defined here, and how it is operated as a sequencer is outside it.
- **Simulation, and the cost estimate that comes with it.** Executing an
  unsubmitted transaction to see what it would do belongs to the wallet API, for
  both public and privacy-preserving transactions, for the reason given in the
  Design Rationale. Nothing in this RFP executes a transaction a caller
  supplies. The node serves the fee market a caller prices against
  (Functionality #67 and #68), not a fee for a given transaction.
- **The other node functions the surveyed chains offer that LEZ lacks**
  ([Appendix: Blockchain API and SDK Ecosystem, section 4.1](../appendix/blockchain-api-sdk-ecosystem.md#41-absent-on-both-targets)).
  A read-only call executes program code, and is simulation by another name: a
  caller reads the state through the account reads and executes on its own side.
  A pre-submission acceptance check is the wallet's own simulation together with
  the typed rejection of Functionality #31. Waiting for a confirmation level is
  composed from the status query (#22) and the block subscription (#40) rather
  than offered as a call that blocks. Batch submission is not required: each
  transaction is submitted on its own.
- **Hosting Logos Core in an application, and reaching a node on another
  machine.** The Logos Core host libraries are a separate component outside this
  RFP, and a node elsewhere is reached through Logos Core peering, which carries
  the module's contract (#80). This RFP defines the functions, not how a call to
  them is carried.
- **The Logos Blockchain node and wallet APIs.** Each is a separate component
  outside this RFP.
- **Reaching inside a node.** A consumer reaches a LEZ node only through the API
  defined here. Whatever a node is made of internally, those interfaces are not
  an API this RFP or any other in this set defines. Every read an integrator
  needs is required of the node above, whichever part of it holds the data
  today.
- **Identifying wallets or users.** The node identifies a caller as Logos Core
  does and no finer. It issues no API keys, keeps no user accounts, and applies
  no quota or billing per user. Which callers a deployment admits, and which
  functions it grants them, is decided in the deployer's Logos Core access
  policy.
- **Extending the program event system.** Events exist from program execution
  through indexer capture to the indexer RPC's `getEvents` and
  `subscribeToEvents`. The FFI exports `query_events` and the module does not,
  and the subscription stops at the RPC. This RFP requires both to reach
  applications (Functionality #7 to #9, #44, #45, #84) and does not change what
  an event is or which events are captured. The other requirements neither
  depend on events nor duplicate them, for two reasons: events do not cover
  execution inside a proof, and their availability depends on how an operator
  configured the event filter.
- **Tracking deposits into private accounts without a viewing key.** Private
  post-states are encrypted and readable only by the viewing key holder, so this
  is foreclosed by the design of the chain rather than by the scope of this RFP.
  A third party sees the amount and receiving address of a public-to-private
  send and nothing after it; a viewing key holder reads the rest through the
  viewing key reads above.
- **Deploying and operating a node.** Mainnet deployment is not part of this
  RFP: whether, when, where and by whom a node exposing this API is deployed or
  operated, which callers a deployment admits, and who holds the administrative
  set are matters for whoever deploys it.

## ⚠ Platform Dependencies

Most capabilities required above are already present on the indexer RPC and
unexported, already computed by the indexer and discarded, or derivable from
data the indexer store already holds. The exceptions are what the target
architecture needs of a node that today's indexer is not.

- **The L1 chain identifier** `query_network_identity` returns. It exists,
  inscribed in the Logos Blockchain genesis block and read at ledger
  initialisation, but no L1 route serves it and the indexer's Bedrock
  configuration carries only an endpoint. Reaching it therefore depends on Logos
  Blockchain exposing it, or on the zone obtaining it at initialisation and
  retaining it. The requirement that an unobtainable value be reported as
  unavailable keeps the rest of the surface deliverable while that is
  outstanding.
- **Peer-to-peer participation outside sequencer mode.** A node relays a
  submission (Functionality #30) and observes the pending set (#35) over the LEZ
  peer-to-peer network, which is not part of this API. Today only the sequencer
  joins that network: transaction gossip runs on a topic per zone in the
  sequencer's gossip actor (`lez/sequencer/actors/gossip/src/actor.rs:336-338`),
  and the indexer joins no peer-to-peer network at all. A proposal states how a
  node outside sequencer mode relays and observes, and on which side of this API
  that work is delivered.
- **Sequencer mode serving this API.** The indexer and the sequencer run today
  as separate services, each with its own RPC, and the requirements marked for
  sequencer mode need a node in that mode to serve the API defined here. A
  sequencer's pending set is also only pushed to today
  (`lez/mempool/src/lib.rs:66-92`), so listing it is work in the sequencer. A
  proposal states how a node in sequencer mode comes to serve this API.
- **Blocks before they are final.** The node reads zone blocks only once they
  are final on L1 (`lez/chain_state/src/zone_indexer.rs:54-86`), so the status
  levels short of final (Functionality #22) need it to learn of a block earlier,
  from the L1 or, in sequencer mode, from the blocks it produces.

Access control rests on Logos Core in four places.

- **Caller identities.** A module sees the runtime and named modules today, and
  some call paths carry no identity. A shell identity and an operator identity
  are in development with the per method grants below. Functionality #91 refuses
  a caller the runtime cannot identify, so the administrative set is reachable
  by a shell or an operator only once those identities exist.
- **Per method grants.** The access policy admits a caller to a module today.
  Narrowing a caller to a list of methods is in development
  ([logos-liblogos#232](https://github.com/logos-co/logos-liblogos/pull/232),
  [logos-capability-module#37](https://github.com/logos-co/logos-capability-module/pull/37))
  and is what Functionality #90 depends on. Functionality #91 does not depend on
  it: the module's own refusal keeps the administrative set closed meanwhile.
- **Callers on another runtime.** A wallet on another runtime reaches the node
  through Logos Core peering, which is also in development
  ([logos-liblogos#230](https://github.com/logos-co/logos-liblogos/pull/230)),
  and which identifies each peered runtime by its own keys. Serving many users'
  runtimes from one node through peering is still in design. A proposal states
  how a per method grant is enforced for such a caller where the platform does
  not enforce it.
- **Events per caller.** A Logos Core module event reaches every subscriber of
  the module. Functionality #95 and Privacy #4 require delivery per caller, so a
  proposal states how a subscription is delivered to the caller that opened it
  and to no other.

## 👤 Recommended Team Profile

This specification is designated for integration into `logos-execution-zone` and
`lez-indexer-module`, so the work sits with people who know both well:

- Rust, and designing a stable C FFI with explicit ownership of allocations
- The LEZ indexer, sequencer, and chain state, and the privacy model of
  privacy-preserving transactions
- Logos Core modules, their contracts, and access policies
- Peer-to-peer networking with libp2p and gossipsub
- Production node software: storage growth, concurrency, and observability

## ⏱ Timeline Expectations

Estimated duration: **16–20 weeks**. Most of the read surface is exposure of
what the indexer already holds, and most of the time goes to what is new: the
persisted effects, transaction status across node modes, the pending set,
subscriptions with resume, access control, and the module contract.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Appendix: Integrating the Logos Technology Stack](../appendix/integrating-logos-technology-stack.md):
  the components the stack exposes, the shapes an integration takes, the
  measured size of the node libraries, and what the stack has not yet settled
- [Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos):
  the target architecture this RFP assumes, with wallet and node modules,
  peering, and module contracts
- [Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md): as-built
  inventory of the LEZ indexer FFI, RPC, and module, the sequencer RPC and FFI,
  `core_ffi` and the `lez_core` module, and the L1 bindings, routes, and module
- [Appendix: Blockchain API and SDK Ecosystem](../appendix/blockchain-api-sdk-ecosystem.md):
  34 API functions across nine established chains, with transports, SDK
  languages, response shapes, and per-function gap notes for LEZ
- [logos-execution-zone](https://github.com/logos-blockchain/logos-execution-zone):
  the LEZ sequencer, indexer, and wallet
- [`logos-execution-zone-module`](https://github.com/logos-blockchain/logos-execution-zone-module):
  `lez_core` Logos Core module
- [lez-indexer-module](https://github.com/logos-blockchain/lez-indexer-module):
  the Logos Core module wrapping the indexer FFI
- [Logos glossary](https://docs.logos.co/get-started/glossary): zone, channel,
  Bedrock, Basecamp, module, viewing key, and note
- [LEZ documentation](https://docs.logos.co/lez): the sequencer and indexer
  roles, and the separation of public and private state
- [Logos Documentation](https://github.com/logos-co/logos-docs)

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
