---
id: RFP-027
title: LEZ Node API
tier: L
status: internal
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

This RFP specifies the LEZ node API, the API the LEZ node module is expected to
expose: the set of functions an integrator needs to follow the chain, track
deposits, confirm transactions, and submit a transaction it has already signed.
Each requirement states what the API provides, and cites the node FFI function
that backs it today where one exists. It is the equivalent of the `eth`
namespace on Ethereum's JSON-RPC: it answers what happened and what the current
state is, and it accepts a signed transaction for the network, holding no keys
and signing nothing.

The requirements are drawn from research into how established chains serve their
integrators and from common practice across the blockchain ecosystem (see
[Why This Matters](#-why-this-matters) and the appendices). The document serves
two purposes:

- **For integrators**, it sets out what the LEZ node API is expected to provide
  before the software is ready, so that they can plan against it and give
  feedback early.
- **For the LEZ maintainers**, it is the basis for implementing the API, and for
  raising any requirement they consider ill-founded or impractical before it is
  built.

This RFP is not an open call for external proposals. The work is to be carried
out by the LEZ maintainers and contributed to the Logos-maintained repositories
of the LEZ node and its Logos Core module, `logos-execution-zone` among them,
and this specification is designated for integration into them through their
usual review and merge process. Once the API is implemented, its reference
documentation supersedes this document for integrators.

### Assumptions

The requirements rest on the assumptions below about where the LEZ node is
heading, drawn from
[Integrate Logos into your application](https://docs.logos.co/build-an-app/integrate-logos)
and from the LEZ code at `dcbd318f0`. Each opens with the assumption itself,
followed by what it is based on and the requirements that rest on it. Each is
for the LEZ maintainers to confirm or invalidate, and one that is invalidated
reopens the requirements that rest on it.

#### A1. Each chain has a wallet module and a node module

Each Logos chain is served by a wallet module and a node module, and a wallet
reaches its node only through the node API, which this RFP defines for LEZ.

The wallet module holds keys, derives, signs, and proves, and runs where the
keys are; everything on that side belongs to the LEZ wallet API, a separate
component outside this RFP. The node module follows the chain, serves its state,
and relays the transactions wallets produce. Today `lez_core` combines the LEZ
wallet with a direct connection to a sequencer.

**Basis:** a wallet and the node it uses can run on separate devices, and be run
by separate parties
([Who uses it, and how it is deployed](#who-uses-it-and-how-it-is-deployed)),
which works only if the wallet reaches its node through the node API alone; the
integration guide draws the split that way. **Rests on it:** the split between
this RFP and the LEZ wallet API throughout, the simulation rationale, and Out of
Scope.

#### A2. A sequencer is a LEZ node in sequencer mode

A LEZ node runs in one of two modes, in sequencer mode or outside it, and a
sequencer is a LEZ node in sequencer mode. It serves the same API as any other
node.

In that mode it also holds the mempool it builds blocks from, produces blocks,
and inscribes them on the Logos Blockchain.

The mode decides what the node depends on. In either mode a LEZ node depends on
a Logos Blockchain node, from which it reads the inscriptions it follows. In
sequencer mode it also depends on a Logos Blockchain wallet module, which holds
the L1 keys it inscribes blocks with; outside sequencer mode that module is not
needed. Logos Core's module builder documents optional dependencies, loaded with
a module when installed and never required (`logos-module-builder`
`docs/configuration.md`, `optional_dependencies`), but whether that serves a
dependency needed in one mode only, and how a node configured for sequencer mode
reports a wallet module that is absent, is open. This RFP defines every function
once. Where a function or an outcome is available only in sequencer mode, or
means something different there, the requirement says so, and a node outside
sequencer mode reports it as not available in its mode ([#99](#f-99)).

**Basis:** the integration guide, which has a LEZ node acting as a sequencer
also use a Logos Blockchain wallet. **Rests on it:** the node's mode in status
([#44](#f-44)), admission and relay ([#45](#f-45) to [#47](#f-47)), the mempool
([#50](#f-50)), and availability by mode ([#99](#f-99)).

#### A3. Following the chain, serving its history, and sequencing are one node

Following the chain, serving its history, and sequencing are functions of one
LEZ node, which serves the API defined here in either mode.

Today the node and the sequencer run as separate services, each with its own
RPC, and the node joins no peer-to-peer network. Bringing them together is part
of the work as far as the API needs it, and Platform Dependencies lists what
that takes.

**Basis:** the integration guide, which draws a single LEZ node. **Rests on
it:** every requirement that a node answers in both modes, and Platform
Dependencies.

#### A4. Every node validates what it relays and keeps the state it validates against

Every LEZ node validates a transaction before relaying it, and keeps the state
of every public account and of the commitment set.

That state is what lets a node validate, and serve this API for any account
rather than for some.

**Basis:** the gossip actor's validation of each transaction it receives, and
the state the node holds today. **Rests on it:** validation at submission
([#46](#f-46) and [#47](#f-47)), and every read of account state, membership
proofs, and publications ([#15](#f-15) to [#38](#f-38)).

#### A5. LEZ nodes talk to each other over a p2p protocol outside this API

LEZ nodes communicate with each other over the LEZ p2p protocol, which is not
part of this API.

Relaying a submitted transaction to the nodes that sequence and gossiping it run
over that network (see [Out of Scope](#out-of-scope)). What a caller can learn
about the node's place in it, such as how many peers it has, is part of this API
([#92](#f-92) and [#93](#f-93)).

**Basis:** the integration guide, and the sequencer's gossip today. **Rests on
it:** relay ([#45](#f-45) and [#47](#f-47)), the observed mempool
([#51](#f-51)), and the peer reads ([#92](#f-92) and [#93](#f-93)).

#### A6. A node outside sequencer mode sees mempool transactions on gossip

A node outside sequencer mode observes mempool transactions on the LEZ
transaction gossip, and serves those it has not yet seen inscribed as its
mempool.

Transactions sequencers gossip travel on a peer-to-peer topic per zone, which
any node following the zone can subscribe to, so a wallet on the user's own node
can still see a mempool transaction that conflicts with one it is building.

**Basis:** the transaction topic of the gossip actor today
(`lez/sequencer/actors/gossip/src/actor.rs:338-340`). **Rests on it:** the
mempool outside sequencer mode ([#50](#f-50) and [#51](#f-51)) and the received
status ([#39](#f-39)).

#### A7. LEZ blocks are not propagated over the LEZ p2p protocol

LEZ blocks are not propagated between LEZ nodes over the LEZ p2p protocol. Every
node, sequencers included, learns of a block only from its inscription on the
Logos Blockchain.

Sequencers learn of each other's blocks through the L1 alone today, gossip
carrying transactions and approvals
(`lez/sequencer/actors/gossip/src/lib.rs:4-6`), and no plan to change that was
found. A sequencer inscribes a LEZ block in an L1 block, and the block's Bedrock
status becomes `finalized` once that L1 block is irreversible, about an hour
after inscription on testnet today.

**Basis:** the code at the commit read, and the absence of any plan found.
**Rests on it:** the transaction statuses ([#39](#f-39)), the subscription to
`pending` blocks ([#59](#f-59)), and the `pending` reads of
[Assumption A8](#a8-every-node-serves-pending-reads-as-well-as-finalized-ones-and-reads-default-to-pending).
**If invalidated:** a node learns of a block before its inscription, and
[#39](#f-39) gains a status for that.

#### A8. Every node serves `pending` reads as well as `finalized` ones, and reads default to `pending`

Every LEZ node answers a read of chain state at one of two Bedrock statuses, and
answers at `pending` unless the caller asks for `finalized`.

The blocks the two statuses see are nested. A `finalized` read sees only the
blocks inscribed in L1 blocks that are final. A `pending` read sees those, and
also the blocks inscribed in L1 blocks that are not yet final. Only those latter
blocks can be replaced, by an L1 reorganisation or a competing inscription. A
transaction's status moves through being received, inscribed, and finalised. The
sequencer service already keeps both sets of blocks and answers at `pending`
(`lez/chain_state/src/chain.rs:1-3`), while the node today reads only to
finality (`lez/chain_state/src/zone_indexer.rs:54-86`).

**Basis:** the sequencer service, which keeps both and answers at `pending`, and
a finality delay too long for a wallet to build on `finalized` reads alone.
**Rests on it:** the Bedrock status a read takes ([#97](#f-97)), the tips every
read reports ([#98](#f-98)), status ([#39](#f-39)), the subscription to
`pending` blocks ([#59](#f-59)), the chain tip ([#80](#f-80)), the fee state
([#88](#f-88) and [#89](#f-89)), and the recovery binding lookup ([#29](#f-29)).

#### A9. The API defined here is the LEZ node module's API

The API this RFP defines is the API the LEZ node module is expected to expose,
and it is defined here in CDDL, in the profile of the draft Logos Core
specification: `spec-module-interface.md`, `spec-cdcddle.md`,
`spec-module-transport.md`, and `spec-module-commitment-model.md`, under
`docs/logos-core/raw/` in `logos-lips`, branch `draft/logos-core-module-specs`
at commit `6cac8335`.

The `.cddl` schema the module delivers is its contract, and the schema root
computed from it is the contract's identity
(`spec-module-interface.md:189-192`). Callers use clients generated from that
schema, whether they run in the same runtime as the node or reach it from
another through Logos Core peering. The CDDL in this document is indicative: it
fixes what each method carries, and the LEZ maintainers finalise names and
encodings ([#96](#f-96)). The requirements below state what the API provides;
the node FFI functions that back each method are cited where they exist today,
and a capability reaches a caller only once the module carries it
([#100](#f-100)). How the module is built, including any FFI library it links,
is the implementation's. See
[Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md) for the
as-built inventory of every layer.

The CDDL profile is a draft specification, not yet implemented in Logos Core.
Modules publish LIDL contracts today, and their values travel as JSON.

**Basis:** the integration guide and the draft Logos Core specification. **Rests
on it:** the module contract ([#96](#f-96)), module parity ([#100](#f-100)),
access control and concurrent callers ([#104](#f-104) to [#111](#f-111)), and
Supportability [#9](#sup-9) and [#10](#sup-10).

#### A10. Each LEZ node follows one zone

Each LEZ node follows a single zone, the one its configuration names, and every
function answers for that zone alone. A deployment that serves several zones
runs a node for each.

The node's configuration names one channel today
(`lez/indexer/core/src/config.rs:37`), and its store is namespaced by channel so
that nodes for different zones can share a storage directory
(`lez/indexer/core/src/lib.rs:108-110`).

How this assumption fits Logos Core is unclear at this stage. Logos Core runs
one instance of each kind of module, so in Basecamp today an application that
supports several zones has to restart the LEZ node module to move from one zone
to another. Running a node per zone would need Logos Core to run several
instances of the node module, one per zone; otherwise one node serves several
zones, and this assumption is invalidated. A clear plan for this is required
(Supportability [#12](#sup-12)).

**Basis:** the node's configuration and store today. **Rests on it:** the
network identity ([#81](#f-81)), storage reset, which removes the store of the
zone the configuration names, and every read, none of which takes a zone. **If
invalidated:** every read takes the zone it answers for, and the network
identity lists the zones the node follows.

### Who uses it, and how it is deployed

**Integrators.** The integration guide names three kinds of integrator, and this
API serves all three:

- developers building applications as Logos modules, run in Basecamp or with
  `logosctl`;
- integrators adding LEZ to their own product or infrastructure, such as an
  exchange, a custodian, or an existing application, by hosting Logos Core;
- node operators offering access to their LEZ nodes as a service, paid or not.

**Deployments.** A wallet module and the node it uses can be run:

- by the same user on the same machine, in one runtime or on one device;
- by the same user on separate machines: a wallet on a phone or a desktop using
  the user's own node on a desktop, a home server, or a VPS;
- within the same infrastructure: a custodian or an exchange running both
  wallets and nodes, its users reaching them through its own interface;
- by separate parties: the mobile wallet of company A using a LEZ node run by
  RPC provider B.

The API is the same in every case. What changes is who calls a node, how many
callers it serves, and what its operator learns about them. A node therefore
scales with its callers and the accounts they follow
([Performance #8](#perf-8)), sorts its functions into categories a deployer
grants separately ([Function categories](#function-categories)), and documents
what its operator can see ([Privacy #6](#priv-6)). Pairing a wallet's runtime
with a remote node, and enrolling and authenticating its callers, are delivered
by Logos Core and are outside this RFP.

**Reference users.** The requirements are written against two. An exchange reads
chain state, tracks deposits to accounts it controls, and waits for a
transaction to reach a status it is willing to act on. A wallet also needs what
an exchange never asks for: the proofs it spends with, the mempool it checks
before proving and submitting, and the publications and encrypted notes it
receives through.

Wallet and key management, transaction construction and signing, the protocol
between LEZ nodes, the Logos Blockchain APIs, and hosting Logos Core in an
application are separate components outside the scope of this RFP. The shapes an
integration takes and what embedding a node costs are described in
[Appendix: Integrating the Logos Technology Stack](../appendix/integrating-logos-technology-stack.md).

## 🔥 Why This Matters

An integrator adding a chain reaches it through a node API, and expects that API
to answer the same questions it answers on the chains the integrator already
supports. The requirements below are therefore drawn from what those integrators
rely on elsewhere. Nine established chains were surveyed for that purpose. LEZ
does not currently meet several of the norms: there are no per-transaction
effects; no status saying how settled a transaction is; and no subscription
reaches the node module, which is the boundary an integrator can actually call.
Some norms this RFP declines rather than fills: simulation, for the reason given
in the Design Rationale, and the functions that go with constructing a
transaction rather than reading the chain, which [Out of Scope](#out-of-scope)
lists one by one. See
[Appendix: Blockchain API and SDK Ecosystem, section 4](../appendix/blockchain-api-sdk-ecosystem.md#4-gap-summary).

Most of the missing capability already exists one layer below the FFI, as the
readiness markers below record, so the work is largely exposure rather than
invention.

## 🏗 Design Rationale

### The target is LEZ testnet 0.4

The node API is delivered for LEZ testnet 0.4 (Supportability [#1](#sup-1)).
Testnet 0.4 is to be released from the LEZ `dev` branch, so the readiness
markers and file references below were read at commit `dcbd318f0` on that
branch, which is newer than the `v0.4.0-rc3` tag. The markers describe the
starting point for the work, not a limit on what the API may express.

Fees are already in force there. A public transaction declares a payer, a gas
limit, a tip, and a cap on the fee
(`lez/indexer/service/protocol/src/lib.rs:291-307`). A privacy-preserving
transaction pays in one of two ways: from a reserve its proof commits to, which
also commits the tip paid from it and names where a surplus is refunded
(`lez/indexer/service/protocol/src/lib.rs:373-380`), or through a fee
declaration signed by one of its public signers, settled as a public
transaction's is (`lez/indexer/service/protocol/src/lib.rs:402-403`,
`lez/chain_state/src/classify.rs:171-183`). Gas is metered in two dimensions,
execution and storage (`lez/programs/fee/core/src/lib.rs:24-29`), and storage is
charged on a transaction's serialized size, a privacy-preserving transaction's
proof being counted at a fixed length (`lez/common/src/transaction.rs:38-57`).
Cost is therefore something a caller estimates, reports, and budgets against.
Each user transaction's outcome is recorded in a receipt the block carries:
whether its action applied, the gas charged in each dimension, the base fee, the
tip, and the refund (`lez/common/src/block.rs:59-105`). Who paid, and by which
of the two ways, is computed when the transaction settles and then discarded.

[Functionality](#functionality) [#7](#f-7) requires the fee to be reported
separately from the balance changes, so a caller reads it rather than
reconstructing it by arithmetic, and [#8](#f-8) requires the gas consumed
alongside it.

### Simulation belongs to the wallet, not to the node

Simulation is the most prominent capability an integrator expects from a node
API that this RFP does not require. Six of the nine surveyed chains execute an
unsubmitted transaction and return its outcome, and an RPC provider elsewhere is
expected to offer it
([Appendix: Blockchain API and SDK Ecosystem, section 1.15](../appendix/blockchain-api-sdk-ecosystem.md#115-simulate-transaction-execution)).
On LEZ that shape does not fit.

The private part of a privacy-preserving transaction is executed by the wallet,
which runs the program locally over notes only it can decrypt. That execution
produces a proof that it was correct, and the proof travels inside the
transaction the wallet submits to the chain. A node verifies the proof but never
runs the private execution itself, and holds neither the witness material nor
the decrypted notes it runs over, so it cannot simulate that part; and by the
time a transaction reaches a node, the wallet has already done the execution a
simulation would repeat. The node executes only the public part. Zcash does not
simulate its shielded leg, for the same reason. It does not simulate its
transparent leg either, though nothing there prevents it, so it is no precedent
for the public half.

Public execution could be simulated by the node, but siting it there would split
one capability across two components and answer against the node's state rather
than the state the caller is building on, its own transactions not yet included
among it. It would also show the node every transaction a caller considered, not
only those it submitted, which a node several wallets share should not see
(Privacy [#5](#priv-5), Privacy [#6](#priv-6)). The wallet already holds the
executor for the private path and can read whatever state it needs through the
queries below, so it can answer for both kinds of transaction, against a state
it chose, at the moment it is constructing the transaction. That is also what
simulation is for elsewhere: Stellar returns the transaction data and the
minimum resource fee that the caller copies back into the transaction before
submitting, which makes simulation a construction step rather than a read.
Construction is out of scope here and belongs to the LEZ wallet API, defined
separately. [Out of Scope](#out-of-scope) records the exclusion.

The LEZ wallet already works this way. Before it sends a public transaction it
simulates it locally against the actor states and program loader state it has
read, measures the gas the run consumed to set the declaration's gas limit, and
refuses a send whose simulation reverts unless the caller allows it
(`lez/wallet/src/lib.rs:238-292`, `lez/wallet/src/lib.rs:2352-2362`,
`lez/wallet/src/lib.rs:2706-2783`). What it needs from a node for that is reads:
the states the transaction touches, which the account reads below serve, and the
current fee market, which it prices the declaration against and reads from the
sequencer service's `getFeeState` today (`lez/wallet/src/lib.rs:363-365`).
[Functionality](#functionality) [#88](#f-88) and [#89](#f-89) require the node
to serve the fee market, and [#97](#f-97) lets every read answer at `pending`,
so a wallet simulating on its own side reaches every input it needs, at the
state it is building on, through the node API.

### Precedents for the wallet and node split

Bitcoin and Ethereum both draw the line this RFP sits on one side of: a wallet
that holds the keys, and a node that holds none and serves the chain.

**Bitcoin: BDK.** The [Bitcoin Development Kit](https://bitcoindevkit.org) keeps
the wallet, `bdk_wallet`, with descriptors, key derivation, coin selection, and
transaction building and signing, apart from four interchangeable chain clients
for Esplora, Electrum, Bitcoin Core's JSON-RPC, and compact block filters. The
wallet performs no network I/O: the integrator fetches chain data with a client
and applies it to the wallet, so the same wallet works against any of them (read
at `bdk-ffi` `3.1.0-alpha.0` and the BDK crates it depends on).

**Ethereum: Geth and viem.** A Geth node serves the `eth` JSON-RPC namespace:
reads of chain state, gas estimation, and `eth_sendRawTransaction` for a
transaction signed elsewhere. [viem](https://viem.sh), the most downloaded
Ethereum library on npm
([Appendix: Blockchain API and SDK Ecosystem, section 3.3](../appendix/blockchain-api-sdk-ecosystem.md#33-deprecations)),
keeps a public client, which reads from a node over HTTP, WebSocket, or an
injected provider, apart from a wallet client and its local accounts, which hold
the keys and sign. A transaction is signed in the library and handed to the node
raw.

**What carries over.** A wallet reaches its node only through the node API, so
the functions below are specified by what they answer, not by where the node
runs or which component holds the data
([Assumption A1](#a1-each-chain-has-a-wallet-module-and-a-node-module)). And a
convenience that syncs a wallet against a node, which BDK leaves to its
integrators and the `lez_core` module ships today, belongs to the LEZ wallet
API, not to this one.

### What a node can answer about a privacy-preserving transaction

**The constraint.** A public account's state is held in the clear by every node,
as part of the state it validates against. A private account's state is on chain
only as a commitment; the state itself travels inside the transaction encrypted,
and only the account's viewing key decrypts it. A node holds no viewing keys.

A privacy-preserving transaction is not opaque in full. A
`PrivacyPreservingMessage` carries, in plaintext, the deliveries that cross
between its private execution and public state in either direction, an optional
public root call, the public nonces it consumes, its validity windows, and its
fee terms (`lez/indexer/service/protocol/src/lib.rs:337-404`). The node executes
that public part itself
(`lee/state_machine/src/validated_state_diff/mod.rs:382-407`), so the public
leg's effects are as readable as a public transaction's, but only from
execution: the body carries the inputs to the public part, not its post-states.
The fee is paid in the clear either way: a fee reserve is one of those
crossings, and it names the address it was paid from
(`lez/chain_state/src/classify.rs:113-128`), and a declared fee names a payer
that signs the transaction.

The private part carries four kinds of encrypted note. Each private action
carries a `nullifier`, a `commitment`, the commitment set `root` it was proven
against, and an `encrypted_post_state`; each async message published sealed
carries a commitment and a note; each recovery binding carries a private address
and a note; and the fee fallback carries a private action of its own
(`lez/indexer/service/protocol/src/lib.rs:362-441`). An async message may
instead be published clear, as a plaintext envelope, at the prover's choice per
message (`lez/indexer/service/protocol/src/lib.rs:392`,
`lez/indexer/service/protocol/src/lib.rs:432-435`,
`lee/state_machine/core/src/circuit_io.rs:292-295`). A node can report that a
private action occurred, prove a commitment's membership, and serve every
ciphertext. What it cannot read is what those ciphertexts hold: which private
account an action touched, and the state it now carries, its nonce and its actor
states, the native balance among them
(`lez/indexer/service/protocol/src/lib.rs:100-127`).

A send to a private account sits between the two. A public-to-private send is
published clear, and a send from a private account may be: its envelope names
the sender, a private one as the prover presents it, the receiving private
address, and the message in plaintext
(`lee/state_machine/core/src/program/mod.rs:314-332`). The amount and the
receiving address of a send published clear are therefore readable, while who
controls that address and what it holds afterwards are not. A send published
sealed reveals neither.

Two limits bound this. Decryption requires the viewing key, which belongs to the
wallet. And a node cannot link a private action's input to its output: the
protocol records that a private action's commitment is not necessarily connected
in content to its nullifier (`lez/indexer/service/protocol/src/lib.rs:366-368`).

**Consequences for the API.** The surface serves the wallet the material it
needs to interpret private state itself, and serves other consumers the public
leg. Commitments, nullifiers, roots, and ciphertext are returned rather than
elided, and each read says which side of the boundary it answers for, so a
consumer is not left to infer the boundary from the shape of a response.

[Functionality](#functionality) [#2](#f-2) requires the returned transaction to
discriminate public and privacy-preserving bodies, so a consumer knows which it
is holding. [#6](#f-6) scopes the balance deltas the transaction read returns to
public transactions and the public leg of privacy-preserving ones, which is what
an integrator crediting deposits reads: a deshield into a public account is
reported, and a transfer between private accounts is not. Because the public leg
of a privacy-preserving transaction is readable only by executing it, [#5](#f-5)
persisting what ingest computed is what makes that leg reportable at all.

The wallet is the other consumer, and there the work is on this surface rather
than on the capability. [#36](#f-36) and [#37](#f-37) require the membership
proofs and the root they are proven against, which a wallet needs to spend. The
publication listing and recovery-binding reads under Private message delivery
are what a wallet needs to receive. The sequencer service answers all of these
today, and its own RPC marks them for removal once the wallet reads from the
node (`lez/sequencer/service/rpc/src/lib.rs:64-65`), while the state the node
already holds carries the data (`lee/state_machine/src/state/mod.rs:357-438`).
What is new is serving them from the node. [#71](#f-71) requires the
oldest-first ordering a wallet walking forward from its last processed block
relies on. Recognising which of what it reads belongs to it, and decrypting it,
is the wallet's.

### Serving what a viewing key opens

**Who holds only a viewing key.** Several consumers need to read a private
account without being able to move what it holds: an auditor, an accountant or
tax provider, a custodian monitoring funds whose spending keys are held offline,
and a member of a shared private account who is given sight of it but not
control. Each holds a viewing key and nothing else. Zcash serves them in two
ways. `z_importviewingkey` hands the node the key, which leaves the node's
operator able to read everything the key can. lightwalletd instead streams
compact blocks ([ZIP 307](https://zips.z.cash/zip-0307)) carrying only what a
light client needs to detect payments to its keys, by height range, and the
client scans them on its own side
([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).

**The node serves; the wallet scans.** The node holds no viewing keys (see the
constraint above) and keeps the state of every account rather than of any one,
so it takes the lightwalletd position. Recovering a particular account's states
from what the chain carries is an active process run for that account: trial
decryption, checking each opened note against the transaction that commits it,
and following the account from there. That is wallet work and belongs to the LEZ
wallet API, as do how a viewing key is exported, how a shared account's viewing
key is derived, and what a viewing key holder can and cannot establish. What the
node owes is the material, complete and unfiltered. Each note is encrypted under
a key derived from a secret shared with the recipient's viewing key and a public
value: the action's nullifier, the sealed message's commitment, or the bound
address (`lee/state_machine/core/src/encryption/mod.rs:91-150`). The walk
therefore carries those values with each note.

**Consequences for the API.** [Functionality](#functionality) under Viewing key
reads requires a walk of every encrypted note with the public values that open
and check it, and nothing a scanner cannot use: the proofs and witness sets a
privacy-preserving transaction carries are omitted. The walk covers every
private account kind, since the node cannot tell them apart, and is
parameterised by position alone, so the walk itself reveals nothing about which
accounts the consumer holds keys for.

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

This RFP therefore sorts the node's functions into categories rather than
defining kinds of account; [Function categories](#function-categories) below
sets them out.

**The boundary is the module, not the FFI.** A caller identity exists only on a
call that arrives through Logos Core. Whoever links the node's library directly
is the node's host and can call whatever it exports, so the requirements below
bind the node module's contract, which is the surface a wallet reaches.

**Callers are no finer than Logos Core makes them.** Two wallets in two modules,
two UI plugins, or two runtimes are two callers. Two wallets behind one calling
module are one caller, and the node cannot tell them apart. Because the node
keeps nothing per wallet, it does not need to. Sessions and user identity are
Logos Core's: a paired runtime or the holder of a token arrives as the caller
identity Logos Core attaches to the call, and the node reports, meters, and logs
against that identity rather than issuing credentials of its own
([Out of Scope](#out-of-scope)).

### Function categories

Every function belongs to one of three categories, and a deployer grants a
category, or any single function in it, to the callers it chooses
([#104](#f-104) and [#105](#f-105)).

- **User**: every function a wallet or an integrator calls to read the chain,
  follow it, and submit a transaction, and to learn whether the node is healthy
  and connected, through its status, version, and peer count. Every deployment
  grants it to the wallets the node serves.
- **Operations**: functions that observe how the node runs in more detail than a
  user needs, or that disclose what an operator would not show its users, such
  as the peer list with other operators' addresses ([#93](#f-93)) and any
  metrics or diagnostics the module exposes. They change nothing, and a deployer
  grants them to its own monitoring.
- **Administrative**: functions that change how the node runs: starting and
  stopping it, resetting its storage, and changing its configuration or its
  logging. They are for the node's operator alone.

**Operations and administrative functions are closed unless granted.**
Enforcement of the access policy is off unless the deployer turns it on, and a
policy in force may admit a caller to every method of the module, so neither
category can rely on the policy to protect it. [#106](#f-106) requires the
module to refuse such a call itself, on the caller identity, unless the caller
is the host runtime, an operator of that runtime, or a caller the node's
configuration names for that category.

### Retention is bounded and reported

A deployment bounds the history the node retains (Reliability [#7](#rel-7)), and
the node reports what it holds rather than leaving a caller to discover it
through a failed request. History is what can be bounded: blocks, transaction
bodies, effects, per-account indexes, and events. The state the node validates
against, the commitment set, the nullifier set, publications, and recovery
bindings among it, only grows (`lee/state_machine/src/state/mod.rs:163-174`),
and the node reports its size rather than bounding it. Cosmos reports
`earliest_block_height` and Bitcoin `pruneheight` for that purpose, and XRPL
reports what it retains as a range that may have gaps
([Appendix: Blockchain API and SDK Ecosystem, section 1.2](../appendix/blockchain-api-sdk-ecosystem.md#12-health-and-sync-status-check)).
EIP-4444, which bounds the history Ethereum clients keep, does so deliberately
so that callers seek old data elsewhere, which only works if a caller can tell
what a node no longer holds
([section 1.29](../appendix/blockchain-api-sdk-ecosystem.md#129-read-historical-state-at-a-past-version)).

[Functionality](#functionality) [#90](#f-90) requires what the node retains to
be exposed, and [#91](#f-91) requires a read below it to be answered as such,
distinctly from a read for data that never existed.

## ✅ Scope of Work

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT,
RECOMMENDED, NOT RECOMMENDED, MAY, and OPTIONAL in this document are to be
interpreted as described in BCP 14
([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119),
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they
appear in all capitals, as shown here. Hard requirements are stated with MUST or
MUST NOT, and soft requirements with SHOULD; text outside the requirement lists,
such as rationale, descriptions of the current code, and citations, is
informative.

### Hard Requirements

#### Functionality

Each requirement carries a readiness marker describing its starting point in the
current LEZ codebase:

- **Ready**: the node FFI already exports the capability, and only a
  specification, test, or documentation obligation remains.
- **Ready, not exposed**: the capability exists in the node service, RPC, or
  store, or the sequencer service serves or computes it from state the node also
  holds, but the node FFI does not export it. The work is exposure.
- **Computed, not persisted**: the node derives the data during ingestion but
  neither stores nor exposes it. The work is persistence plus exposure.
- **New**: the capability does not exist today, whether or not the data it would
  serve does.

A requirement whose behaviour depends on the node's mode says so. The markers
indicate effort, not priority: every hard requirement is required regardless of
its marker. The CDDL under each function is indicative
([Assumption A9](#a9-the-api-defined-here-is-the-lez-node-modules-api)): it
fixes what the method carries, while the requirements underneath it state what
is required. The CDDL of each method whose requirements carry a marker opens
with a `; readiness:` comment summarising those markers, with requirement
numbers where they differ; CDDL comments are presentation only and not part of
the schema.

The markers describe the FFI on the LEZ `dev` branch, so a marker of **Ready**
means ready in the FFI. Only the `v0.4.0` release candidates (`v0.4.0-rc1` to
`v0.4.0-rc3`) are considered at the time of writing. Where a capability landed
on `dev` after `v0.4.0-rc3`, as transaction receipts did, the marker gives its
state at the release tags and a note says it is on `dev`. A capability ready in
the FFI reaches a caller only once the module carries it, which [#100](#f-100)
requires.

##### Shared types

The CDDL in this section follows the profile of
[Assumption A9](#a9-the-api-defined-here-is-the-lez-node-modules-api). It opens
the module's single `.cddl` file, and the rules under each function below
continue it. Every method is a pair of maps, `lez_node.<method>_request` and
`lez_node.<method>_response`, and every event a `lez_node.<name>_event` map. The
rules below are shared by them.

```cddl
; -- metadata --
_module = "lez_node_module"

; -- identifiers and values --
lez_node.hash = bstr .size 32
lez_node.account_id = bstr .size 32
lez_node.zone_id = bstr .size 32
lez_node.commitment = bstr .size 32
lez_node.nullifier = bstr .size 32
lez_node.selector = bstr .size 8
lez_node.block_id = uint64
lez_node.timestamp = uint64
lez_node.position = uint64

; u128 values: 16 bytes, little-endian
lez_node.amount = bstr .size 16
lez_node.nonce = bstr .size 16

; a block position: its identifier and its hash together
lez_node.block_ref = {
    id: lez_node.block_id,
    hash: lez_node.hash,
}

; where a caller resumes: the last block it processed, its hash optional
lez_node.resume_from = {
    id: lez_node.block_id,
    ? hash: lez_node.hash,
}

; a resume position whose hash the node's chain does not match; `common` is
; the last block on the node's chain that the caller's position shares
lez_node.resume_mismatch = {
    common: lez_node.block_ref,
    tips: lez_node.tips,
}

; -- what a read answers at --
lez_node.bedrock_status = "pending" / "finalized"

; the last block visible at each Bedrock status
lez_node.tips = {
    ? pending: lez_node.block_ref,
    ? finalized: lez_node.block_ref,
}

lez_node.served_at = {
    bedrock_status: lez_node.bedrock_status,
    ? block: lez_node.block_ref,
    tips: lez_node.tips,
}

; -- pages --
lez_node.order = "oldest_first" / "newest_first"
lez_node.limit = uint32
lez_node.cursor = bstr .size (1..256)

; -- subscriptions --
lez_node.subscription_id = bstr .size 16

lez_node.subscription_end = {
    reason: "unsubscribed" / "consumer_lagging" / "node_stopped",
    ? last_delivered: lez_node.block_ref,
}

; -- errors --
lez_node.error_code =
    "invalid_argument" /
    "too_large" /
    "below_retention_floor" /
    "lagging" /
    "stalled" /
    "backend_failure" /
    "not_available_in_mode" /
    "refused" /
    "node_stopped"

lez_node.error = {
    code: lez_node.error_code,
    message: tstr .size (1..512),
    ? tips: lez_node.tips,
}

; -- transactions --
lez_node.transaction = {
    kind: "public" / "privacy_preserving",
    encoded: bstr,
}

lez_node.actor = {
    account_id: lez_node.account_id,
    program_account_id: lez_node.account_id,
}

lez_node.actor_state = {
    program_account_id: lez_node.account_id,
    data: bstr,
}

lez_node.account = {
    nonce: lez_node.nonce,
    actor_states: [* lez_node.actor_state],
}

lez_node.encrypted_note = {
    ciphertext: bstr,
    ephemeral_key: bstr,
}

lez_node.recovery_binding = {
    address: lez_node.account_id,
    note: lez_node.encrypted_note,
}

lez_node.publication = {
    kind: "clear",
    position: lez_node.position,
    from: lez_node.actor,
    to: lez_node.actor,
    message: bstr,
} / {
    kind: "sealed",
    position: lez_node.position,
    commitment: lez_node.commitment,
    note: lez_node.encrypted_note,
}

lez_node.gas = {
    execution: uint64,
    storage: uint64,
}

lez_node.receipt = {
    applied: bool,
    gas: lez_node.gas,
    base_fee: lez_node.amount,
    tip: lez_node.amount,
    refund: lez_node.amount,
}

lez_node.event_record = {
    block: lez_node.block_ref,
    transaction_index: uint32,
    transaction_hash: lez_node.hash,
    event_index: uint32,
    program_account_id: lez_node.account_id,
    selector: lez_node.selector,
    data: bstr,
}
```

- **Integers above 64 bits.** The profile has no 128-bit integer, so an amount
  or a nonce, a `u128` in LEZ, is a 16-byte string holding the value
  little-endian (`lez_node.amount`, `lez_node.nonce`).
- **Block positions.** A `lez_node.block_ref` names a block by its identifier
  and hash together. The API MUST report one for every tip, since an identifier
  read at `pending` names a height rather than a block ([#23](#f-23)). A caller
  resuming from a block passes a `lez_node.resume_from`, whose hash is optional,
  and the API MUST answer a hash the node's chain does not match with a
  `lez_node.resume_mismatch` rather than resume from it ([#61](#f-61)).
- **Bedrock status and tips.** A read of chain state takes an optional
  `bedrock_status`: `pending` (the default) or `finalized` ([#97](#f-97)). Its
  response carries a `lez_node.served_at` with the status it answered at, the
  block it answered at, and both tips ([#98](#f-98)). Other reads carry the tips
  alone. A tip is the last block visible at its status. The API MUST omit a tip,
  or the block answered at, only while no block is visible at that status.
- **Absence and errors.** The API MUST report a record the node does not hold as
  an omitted field, not an error. Every response MUST be either its success map
  or an `error` map whose `code` is one of `lez_node.error_code`, the expected
  failures this RFP names ([#99](#f-99), [#101](#f-101) to [#103](#f-103),
  [#108](#f-108), [#111](#f-111)). These are contract responses: the shared
  `logos.error_code` values are reserved for failures of the invocation itself,
  such as a request that does not match its schema, and the API MUST NOT use
  them for an expected failure, as the profile forbids
  (`spec-module-interface.md:345-346`, `:3260-3264`).
- **Transactions.** A `lez_node.transaction` carries a transaction as the bytes
  the LEZ `LeeTransaction` type serialises to under Borsh
  (`lez/common/src/transaction.rs:14-35`), the value the sequencer RPC's
  `sendTransaction` takes today (`lez/sequencer/service/rpc/src/lib.rs:52-53`),
  with a `kind` naming which of the two kinds it is. Those bytes are the
  transaction as the chain holds it, so a caller recomputes its hash from them
  ([#3](#f-3)), and decoding them is the caller's.
- **Pages.** A `lez_node.cursor` MUST be stable ([#68](#f-68)); what it holds is
  the implementer's choice. A page MUST carry the position or cursor the next
  page resumes from, and only an empty page marks the end ([#72](#f-72)).
- **Subscriptions.** A subscription is a method that watches, as the profile
  models one: its invocation stays open while it delivers `_event` maps to the
  caller that opened it, and its final response, carrying a
  `lez_node.subscription_end`, ends it (`spec-module-interface.md:419-421`,
  `spec-module-runtime.md:1173-1201`). The caller names each subscription with a
  random `lez_node.subscription_id` of its choosing, which every event carries,
  since an identifier the node returned only in the final response could not
  tell earlier events apart (`spec-module-transport.md:833`,
  `spec-module-runtime.md:1190-1191`).

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

```cddl
; readiness: Ready (#1, #2, #9); Computed, not persisted (#5 to #8); New (#3, #4)
lez_node.query_transaction_request = {
    transaction_hash: lez_node.hash,
}

lez_node.query_transaction_response = {
    ? record: lez_node.transaction_record,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

; option C or D of the shared-hash problem adds the block and index here
lez_node.transaction_record = {
    transaction: lez_node.transaction,
    effects: lez_node.effects,
}

lez_node.effects = {
    diffs: [* lez_node.state_diff],
    balance_deltas: [* lez_node.balance_delta],
    fee: lez_node.fee_report,
    gas: lez_node.gas_report,
    events: [* lez_node.event_record],
    ? root_program_account_id: lez_node.account_id,
}

lez_node.state_diff = {
    stage: "fee_reserve" / "action" / "refund",
    accounts: [* lez_node.account_change],
    commitments: [* lez_node.commitment],
    nullifiers: [* lez_node.nullifier],
    publications: [* lez_node.publication],
    recovery_bindings: [* lez_node.recovery_binding],
}

lez_node.account_change = {
    account_id: lez_node.account_id,
    pre_nonce: lez_node.nonce,
    post_nonce: lez_node.nonce,
    actor_states: [* lez_node.actor_state_change],
}

; an actor state that did not exist, or no longer does, is empty
lez_node.actor_state_change = {
    program_account_id: lez_node.account_id,
    pre: bstr,
    post: bstr,
}

lez_node.balance_delta = {
    account_id: lez_node.account_id,
    program_account_id: lez_node.account_id,
    pre: bstr,
    post: bstr,
}

lez_node.fee_report = {
    kind: "exempt",
} / {
    kind: "declared",
    payer: lez_node.account_id,
    charged: lez_node.amount,
    base_fee: lez_node.amount,
    tip: lez_node.amount,
    refund: lez_node.amount,
} / {
    kind: "reserve",
    payer: lez_node.account_id,
    charged: lez_node.amount,
    base_fee: lez_node.amount,
    tip: lez_node.amount,
    surplus: lez_node.surplus,
}

lez_node.surplus = {
    fate: "none",
} / {
    fate: "refunded",
    amount: lez_node.amount,
    to: lez_node.account_id,
} / {
    fate: "private_credit",
    amount: lez_node.amount,
    alias: lez_node.account_id,
} / {
    fate: "kept_as_tip",
    amount: lez_node.amount,
}

lez_node.gas_report = {
    kind: "exempt",
} / {
    kind: "metered",
    gas: lez_node.gas,
}
```

Returns the transaction carrying the given hash, as stored in the block that
holds it, together with its effects ([#5](#f-5) to [#9](#f-9)). Absence is
signalled by an omitted `record`, not by an error.

1. <a id="f-1"></a>The query MUST return the stored transaction for a known hash
   and an omitted `record` for a hash no indexed block holds, keeping the two
   distinct from a backend failure. **[Ready]**
2. <a id="f-2"></a>The returned transaction MUST discriminate public and
   privacy-preserving bodies, exposing for each the fields the wire type carries
   (`lez/indexer/service/protocol/src/lib.rs:254-441`). A program is deployed by
   a public transaction to the program loader rather than by a transaction kind
   of its own. **[Ready]**
3. <a id="f-3"></a>The returned transaction MUST carry every field of the
   transaction as the chain holds it, so a caller can recompute its hash. The
   wire type drops a privacy-preserving transaction's program image claims
   (`lez/indexer/service/protocol/src/convert.rs:599-602`), so a transaction
   carrying any cannot be reproduced from what is returned. **[New]**
4. <a id="f-4"></a>**Problem: two transactions can share a hash.** Every block
   ends with a fee transaction that carries no nonce and no block identifier
   (`lez/common/src/transaction.rs:355-395`), so when consecutive blocks have
   the same producer and no fee revenue, their fee transactions are identical
   byte for byte and have the same hash. The node's index from transaction hash
   to block keeps only the latest of them
   (`lez/storage/src/indexer/write_atomic.rs:40-46`), and the transaction a read
   by hash returns carries no block identifier, so such a read is ambiguous. The
   LEZ maintainers MUST choose one of the following, and the API documentation
   MUST state the guarantee that results:
   - **A. Make the fee transaction unique.** The fee transaction carries its
     block identifier, as the clock transaction that closes each block already
     does (`lez/common/src/transaction.rs:267-283`), so a hash names one
     transaction everywhere. This changes what a block contains, so it lands
     before testnet 0.4 is frozen.
   - **B. Keep system transactions out of hash lookups.** Transactions the
     sequencer injects, the fee and clock transactions among them, are served
     only as part of their block. A read by hash answers for user transactions,
     which their nonces and nullifiers keep unique.
   - **C. Answer every read by hash with its block.** A read by hash returns the
     block identifier and index of the transaction it found, and every
     transaction is readable by its block identifier and index, which no other
     shares. A hash can still name more than one transaction.
   - **D. Index every occurrence.** The index keeps every block a hash appears
     in, and a read by hash returns each match with its block, for the caller to
     choose. **[New]**

**Effects.** The same read returns what the transaction did, read from storage
rather than recomputed on the caller's behalf, as Solana's `getTransaction`,
XRPL's `tx`, and Sui's `sui_getTransactionBlock` return effects with the
transaction rather than from a call of their own. If returning effects with
every transaction read proves a performance bottleneck, they MAY instead be
served by a separate function, such as
`query_transaction_effects(transaction_hash)`, and the requirements below hold
either way.

5. <a id="f-5"></a>The API MUST return, with a transaction read by its hash, the
   state diff that transaction produced: the accounts whose public state
   changed, with their pre-state and post-state nonces and, for each actor state
   that changed and only those, its pre-state and post-state value; the new
   commitments; the new nullifiers; the publications produced, with their
   positions; the recovery bindings introduced; and the program the public root
   call invokes, absent when execution starts privately. A diff as computed
   today carries each touched account's whole post-state, untouched actor states
   included (`lee/state_machine/src/validated_state_diff/mod.rs:33-42`), and any
   party can write actor states of any size onto any account, so reporting whole
   states would let a third party inflate every effects response for an account
   it targets. The diff MUST be the one the chain applied when the transaction
   settled, as the node computes it on ingest
   (`lez/chain_state/src/apply.rs:391-812`), not one recomputed for the caller.
   A transaction paying a declared fee settles as separate diffs for its fee
   reserve, its action, and its refund (`lez/chain_state/src/apply.rs:654-812`);
   one paying from a reserve its proof commits to carries the reserve as a
   crossing inside its action, with any refund separate
   (`lez/chain_state/src/apply.rs:485-592`). Each diff MUST be reported with the
   state it applied to. **[Computed, not persisted]**

6. <a id="f-6"></a>The read MUST return a balance delta per affected account,
   expressed as pre-state and post-state values, for public transactions and for
   the public leg of privacy-preserving transactions. The native balance is the
   account's actor state at the native token program
   (`lez/indexer/service/protocol/src/lib.rs:106-127`), and a token held through
   another program is that program's actor state, so the delta MUST be reported
   per actor state rather than as a single balance. **\[Computed, not
   persisted\]**

7. <a id="f-7"></a>The fee MUST be reported separately from the balance changes
   the transaction itself caused, naming the account charged, the amount taken,
   and any amount refunded. An exchange reconciling a withdrawal has to tell
   what it sent from what it paid to send it, and a caller that has to
   reconstruct the fee by arithmetic on the remaining deltas is reimplementing
   the chain's fee rules. A public transaction declares a payer, a gas limit, a
   tip, and a cap on the fee; the payer need not be an account the transaction
   otherwise acts on, but it must sign (`lee/state_machine/src/fees.rs:34-52`).
   A privacy-preserving transaction pays either from a reserve its proof commits
   to, together with a tip it commits, or through a fee declaration signed by
   one of its public signers, as a public transaction does, and the effects
   query MUST report which. A surplus left in a proof-committed reserve has one
   of three fates: refunded to the target the proof names, published to the
   payer's bound alias as a private credit, or, when it is no more than
   `MIN_PRIVATE_REFUND` or there is nowhere to send it, kept as tip
   (`lez/chain_state/src/apply.rs:575-592`), and the query MUST report which and
   where. The base fee, tip, and refund, the account charged, which way a
   privacy-preserving transaction paid, and the fate of its surplus are computed
   when the transaction settles and then discarded **\[Computed, not
   persisted\]** (on `dev` since `f99ba435f`, after `v0.4.0-rc3`, the base fee,
   tip, and refund are kept in the block's receipts). A fee-exempt system
   transaction MUST be reported as exempt rather than as one that paid nothing.

8. <a id="f-8"></a>The gas a transaction consumed MUST be reported alongside the
   fee, in each dimension the chain meters, execution and storage
   (`lez/programs/fee/core/src/lib.rs:24-29`), so a caller comparing what it
   budgeted against what it paid can tell a transaction that ran long from one
   that paid a high price per unit. Both kinds of transaction are metered: a
   privacy-preserving transaction is charged a fixed verification cost, storage
   on its serialized size with its proof counted at a fixed length, and the
   metered execution of its public part
   (`lez/chain_state/src/apply.rs:515-544`). The API MUST report a fee-exempt
   transaction as exempt rather than as one that consumed zero, which a charged
   transaction never does, since it always meters storage. **\[Computed, not
   persisted\]** (on `dev` since `f99ba435f`, after `v0.4.0-rc3`, the block's
   receipts carry both dimensions and record an exempt transaction as all zeros,
   `lez/common/src/block.rs:59-95`).

9. <a id="f-9"></a>The effects MUST include the events the transaction emitted,
   in the order it emitted them, each as [#12](#f-12) describes, as Ethereum's
   receipt carries its logs and Solana's `getTransaction` its log messages. The
   FFI serves them today through `query_events` with a transaction hash
   (`lez/indexer/ffi/src/api/query.rs:553-583`), which reports a hash it does
   not hold as an invalid argument; with the transaction read, an unknown hash
   is not found ([#1](#f-1)). **[Ready]**

```cddl
; readiness: New
lez_node.query_transaction_receipt_request = {
    transaction_hash: lez_node.hash,
}

lez_node.query_transaction_receipt_response = {
    result: "receipt",
    receipt: lez_node.receipt,
    tips: lez_node.tips,
} / {
    result: "no_receipt",
    tips: lez_node.tips,
} / {
    result: "not_found",
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}
```

Returns the receipt the block records for a transaction: whether its action
applied, and the gas, base fee, tip, and refund it was charged.

10. <a id="f-10"></a>The API MUST return a transaction's receipt, exporting what
    the node RPC's `getTransactionReceipt` serves
    (`lez/indexer/service/rpc/src/lib.rs:100-105`,
    `lez/indexer/ffi/src/api/query.rs:359-407`). The wallet already reads a
    finished transaction's outcome this way, from the sequencer service today
    (`lez/sequencer/service/rpc/src/lib.rs:92-97`). **[New]** (exported on `dev`
    since `f99ba435f`, after `v0.4.0-rc3`).
11. <a id="f-11"></a>A transaction the chain records without a receipt, the fee
    and clock transactions that close each block
    (`lez/common/src/block.rs:100-105`), MUST be reported as having none,
    distinctly from a hash no indexed block holds. Both return an absent receipt
    today. **[New]**

```cddl
; readiness: Ready (#12, #14); New (#12, #13)
lez_node.query_events_request = {
    from_block: lez_node.block_id,
    ? to_block: lez_node.block_id,
    ? program_account_id: lez_node.account_id,
    ? selector: lez_node.selector,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_events_response = {
    events: [* lez_node.event_record],
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns the program events emitted in a block range, from `from_block` to
`to_block`, defaulting to the tip at the requested Bedrock status, optionally
filtered by emitting program and selector. The events of one transaction come
with the transaction read instead ([#9](#f-9)), so the query takes no
transaction hash, which the FFI's `query_events` does today
(`lez/indexer/ffi/src/api/query.rs:536-543`).

12. <a id="f-12"></a>The query MUST return each matching event record with its
    block identifier, transaction index, transaction hash, emitting program,
    selector, and data payload **[Ready]**, and its index among the
    transaction's events, so a consumer deduplicates and resumes by event
    **[New]**.
13. <a id="f-13"></a>A request the query cannot answer MUST be reported as the
    outcome [#101](#f-101) and [#103](#f-103) define for its case rather than as
    an empty result: a range spanning more than `MAX_EVENT_QUERY_BLOCK_SPAN`
    blocks, or inverted, as an invalid argument; a bound past the tip at the
    requested Bedrock status as the lagging case; a range outside the node's
    event-filter history as data the node does not hold, as a read below the
    retention floor is (the `invalid_argument`, `lagging`, and
    `below_retention_floor` errors). Each returns `InvalidArgument` today
    (`lez/indexer/ffi/src/api/query.rs:553-633`). The query MUST also bound the
    size of its response, as the node RPC does at 8 MiB
    (`lez/indexer/service/src/service.rs:28`), and MUST report a result past the
    bound as too large, the `too_large` error, rather than truncating it.
    **[New]**
14. <a id="f-14"></a>Events emitted by the public part of a privacy-preserving
    transaction MUST be indexed as a public transaction's are
    (`lee/state_machine/src/validated_state_diff/mod.rs:586-621`), and execution
    inside the proof emits none a node can observe, so the query, and the events
    a transaction read returns, answer for public execution only. **[Ready]**

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

**What an account is.** An account identifier can name any account, a program's
included: a program is an account written through the program loader, and its
image sits in that account's actor state at the program loader
(`lee/state_machine/src/state/mod.rs:405-430`). An account holds a nonce and one
actor state per program that has written to it, keyed by that program's account
identifier (`lez/indexer/service/protocol/src/lib.rs:100-110`). A read of one
program's actor state on an account, `query_account_view` below, is how a caller
reads the native balance (the actor state at the native token program), a
balance held through another program, or a program's image (the actor state at
the program loader). It is also the safe way to read any one of them: anyone can
write actor states of any size onto any account
(`lez/indexer/service/protocol/src/lib.rs:136-146`), so a read of the whole
account can be inflated by a third party, as Ethereum's `eth_getStorageAt` reads
one storage slot rather than an account's whole storage.

Every read in this section answers at the Bedrock status the caller chooses,
`pending` or `finalized` ([#97](#f-97)).

```cddl
; readiness: Ready
lez_node.query_account_request = {
    account_id: lez_node.account_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_account_response = {
    ? account: lez_node.account,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns the account record as it stands at the requested Bedrock status: its
nonce, and one actor state per program holding state on the account.

15. <a id="f-15"></a>The query MUST return the account's nonce and its actor
    states, each keyed by the account identifier of the program it belongs to
    (`lez/indexer/service/protocol/src/lib.rs:100-110`). The native balance is
    the actor state at the native token program, and the API documentation MUST
    state how to read it. **[Ready]**
16. <a id="f-16"></a>An identifier the state holds no record for MUST return an
    omitted `account`, distinct from a recorded account whose actor states are
    empty and from a backend failure. **[Ready]**

```cddl
; readiness: Ready, not exposed
lez_node.query_account_view_request = {
    account_id: lez_node.account_id,
    program_account_id: lez_node.account_id,
    ? block_id: lez_node.block_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_account_view_response = {
    ? view: lez_node.account_view,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

; `data` is omitted when the program holds no actor state on the account
lez_node.account_view = {
    nonce: lez_node.nonce,
    ? data: bstr,
}
```

Returns an account's nonce and the actor state of one program, optionally pinned
to a block.

17. <a id="f-17"></a>The API MUST expose a read of one program's actor state on
    an account, with the account's nonce, at the tip or at a pinned block,
    exporting the existing `getAccountView` and `getAccountViewAtBlock` node RPC
    methods (`lez/indexer/service/rpc/src/lib.rs:83-91`). Nothing bounds how
    many actor states an account holds or how large each is, and any party can
    write its own onto any account
    (`lez/indexer/service/protocol/src/lib.rs:136-146`), so a whole-account read
    is not a safe way to read one balance; this is. **[Ready, not exposed]**

```cddl
; readiness: Ready, not exposed (#18); New (#19)
lez_node.query_account_summary_request = {
    account_id: lez_node.account_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_account_summary_response = {
    ? summary: lez_node.account_summary,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

; `balance` is omitted when the native actor state is not a canonical encoding
lez_node.account_summary = {
    nonce: lez_node.nonce,
    ? balance: lez_node.amount,
    actor_states: [* lez_node.actor_state_size],
}

lez_node.actor_state_size = {
    program_account_id: lez_node.account_id,
    size: uint64,
}
```

Returns an account's nonce, its native balance, and the size of each actor state
rather than its bytes.

18. <a id="f-18"></a>The API MUST expose an account summary: the nonce, the
    native balance, and the size of each actor state rather than its bytes,
    exporting the existing `getAccountSummary` node RPC method, so a caller
    learns which programs hold state on an account in a response whose size
    follows their count. **[Ready, not exposed]**
19. <a id="f-19"></a>An identifier the state holds no record for MUST return an
    omitted `summary`, on the same terms as the account read ([#16](#f-16)). The
    RPC method returns a default summary for it today
    (`lez/indexer/service/src/service.rs:151-161`), which a caller cannot tell
    from a recorded account that is empty. **[New]**

```cddl
; readiness: Ready, not exposed (#20); New (#21)
lez_node.query_balances_request = {
    account_id: lez_node.account_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_balances_response = {
    ? balances: lez_node.balances,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.balances = {
    native: lez_node.amount,
    tokens: [* lez_node.token_balance],
}

lez_node.token_balance = {
    program_account_id: lez_node.account_id,
    token: lez_node.account_id,
    amount: lez_node.amount,
}
```

Returns a public account's balances, decoded: its native balance, and its
balance in each token a builtin token program issues.

20. <a id="f-20"></a>The API MUST return a public account's native balance,
    decoded from its actor state at the native token program, as the summary
    already decodes it (`lez/indexer/service/src/service.rs:164`) and as the
    sequencer service's `getAccountBalance` serves it today
    (`lez/sequencer/service/rpc/src/lib.rs:82`). **[Ready, not exposed]**
21. <a id="f-21"></a>The same call MUST return the account's balance in each
    token a builtin token program issues, decoded, each with the identifier of
    the token it is denominated in, and the API documentation MUST state which
    programs it decodes. A wrapped token's balance sits not on the holder's
    account but in a holding account the program derives from the token and the
    holder, whose data is the balance
    (`lez/programs/wrapped_token/core/src/lib.rs:171-198`), so the node MUST
    resolve it for each wrapped token deployed. Solana's node does the same for
    a fixed set of well-known programs, SPL Token among them, and returns any
    other program's accounts as raw bytes
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.8](../appendix/blockchain-api-sdk-ecosystem.md#18-get-a-balance)).
    A balance held through any other program is likewise left to the caller,
    whose SDK decodes the actor state from the program's IDL. **[New]**

```cddl
; readiness: Ready, not exposed (#22); New (#23)
lez_node.query_account_at_block_request = {
    account_id: lez_node.account_id,
    block_id: lez_node.block_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_account_at_block_response = {
    ? account: lez_node.account,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns the account record as it stood at a given block, rather than at the tip.

22. <a id="f-22"></a>The API MUST expose an account read pinned to a block
    identifier, returning the account record as it stood at that block, on the
    same terms as the read at the tip. This exports the existing
    `getAccountAtBlock` node RPC method
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[Ready, not exposed]**
23. <a id="f-23"></a>The pinned read MUST return the hash of the block it
    answered against, and the API documentation MUST state that a block
    identifier read at `pending` names a height rather than a block: until the
    block at it is `finalized`, the same identifier can resolve to a different
    block after that block is replaced, and the hash is what tells a caller
    which one it read. **[New]**

```cddl
; readiness: New
lez_node.query_accounts_request = {
    account_ids: [* lez_node.account_id],
    shape: lez_node.batch_shape,
    ? block_id: lez_node.block_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.batch_shape = {
    kind: "account",
} / {
    kind: "summary",
} / {
    kind: "view",
    program_account_id: lez_node.account_id,
}

lez_node.query_accounts_response = {
    entries: [* lez_node.batch_entry],
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

; an entry omits its record for an identifier the state holds no record for
lez_node.batch_entry = {
    kind: "account",
    ? account: lez_node.account,
} / {
    kind: "summary",
    ? summary: lez_node.account_summary,
} / {
    kind: "view",
    ? view: lez_node.account_view,
}
```

Returns one result per requested account identifier, optionally pinned to a
block.

24. <a id="f-24"></a>The API MUST expose a batch account read taking a list of
    account identifiers and returning a result per identifier in request order.
    An identifier the state holds no record for MUST be reported absent in its
    entry rather than failing the call. **[New]**
25. <a id="f-25"></a>The batch account read MUST accept an optional block
    identifier that pins every account in the batch to the same block, so a
    multi-account read is internally consistent under concurrent ingestion.
    **[New]**
26. <a id="f-26"></a>The batch read MUST also return, per account, the summary
    of [#18](#f-18) or the view of one program's actor state of [#17](#f-17) in
    place of the whole record, as the caller asks, so a batch never carries the
    unbounded actor states [#17](#f-17) warns of. The wallet reads its accounts'
    nonces in a batch today, through the sequencer service's `getAccountsNonces`
    (`lez/sequencer/service/rpc/src/lib.rs:98-102`). **[New]**

##### Private message delivery

Reading the messages a private account receives. A send to a private account the
sender does not hold is published rather than applied, at a position in the
commitment set: as a clear publication whose envelope is in plaintext, always
for a public-to-private send and at the sender's choice for a private one, or as
a sealed publication. The receiving wallet reads the publications, finds those
addressed to it, and later consumes them. Among the surveyed chains only Zcash,
whose shielded payments a recipient finds by scanning, has a comparable
structure. The sequencer service serves it today
(`lez/sequencer/service/rpc/src/lib.rs:112-129`), and the node holds the data in
its own state (`lee/state_machine/src/state/mod.rs:357-406`).

```cddl
; readiness: Ready, not exposed (#27); New (#28)
lez_node.query_publications_request = {
    from_position: lez_node.position,
    limit: lez_node.limit,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_publications_response = {
    publications: [* lez_node.publication],
    recovery_bindings: [* lez_node.recovery_binding],
    next_position: lez_node.position,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns a page of publications in commitment-set order, from a given position.

27. <a id="f-27"></a>The API MUST list publications in position order from a
    given position, each with its position: for a clear publication its
    plaintext envelope, and for a sealed one its commitment and encrypted note.
    Each page MUST also carry, once, the recovery note bound to each destination
    of a clear publication in it, as the state already pages them
    (`lee/state_machine/src/state/mod.rs:178-181`,
    `lee/state_machine/src/state/mod.rs:383-406`) and the sequencer service's
    `getPublications` returns them. **[Ready, not exposed]**
28. <a id="f-28"></a>The listing MUST be bounded, on the same terms as the other
    paginated reads, and MUST NOT take an account identifier, so a wallet
    receiving messages discloses nothing beyond the position it resumes from.
    **[New]**

```cddl
; readiness: Ready, not exposed (#29); New (#30)
lez_node.query_recovery_binding_request = {
    address: lez_node.account_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_recovery_binding_response = {
    ? note: lez_node.encrypted_note,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns the encrypted note bound to a private address, if one is.

29. <a id="f-29"></a>The API MUST return the recovery note bound to a private
    address, or an omitted `note` when none is, at the requested Bedrock status
    ([#97](#f-97)), exporting what the sequencer service's `getRecoveryBinding`
    serves today at `pending`. A wallet building a transaction reads it at
    `pending`: it checks whether each address it would bind is bound already, a
    duplicate binding making the transaction invalid, and whether the
    destination of a clear send is bound, a clear send to an unbound one being
    refused (`lez/wallet/src/lib.rs:1474-1520`). **[Ready, not exposed]**
30. <a id="f-30"></a>The API documentation MUST state that this lookup names a
    private address to the node: the caller's own, or, when a sender checks a
    destination before a clear send, the address of the account it is about to
    pay. A consumer reading its own notes without disclosing its addresses reads
    them from the publication listing or the viewing key walk instead, neither
    of which takes an address. **[New]**

##### Viewing key reads

Serving every encrypted note the chain carries, without the node holding any
key, to a consumer that scans them on its own side (see
[Serving what a viewing key opens](#serving-what-a-viewing-key-opens)). The
publication listing above covers received messages; this walk covers every
encrypted note, account state updates included. Scanning it is the wallet's.

- **Zcash**: `z_importviewingkey` imports the key into the node's wallet, which
  then answers for it; lightwalletd instead streams compact blocks a light
  client scans on its own side ([ZIP 307](https://zips.z.cash/zip-0307)).
- **Ethereum, Solana, Bitcoin**: no equivalent, there being no encrypted state
  to scan.

```cddl
; readiness: New
; `from` is the last block the caller processed; omitted, the walk starts
; at the retention floor
lez_node.query_private_notes_request = {
    ? from: lez_node.resume_from,
    limit: lez_node.limit,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_private_notes_response = {
    blocks: [* lez_node.note_block],
    ? next_from: lez_node.block_ref,
    served: lez_node.served_at,
} / {
    mismatch: lez_node.resume_mismatch,
} / {
    error: lez_node.error,
}

lez_node.note_block = {
    block: lez_node.block_ref,
    transactions: [* lez_node.note_transaction],
}

lez_node.note_transaction = {
    transaction_hash: lez_node.hash,
    index: uint32,
    applied: bool,
    notes: [* lez_node.private_note],
}

lez_node.private_note = {
    kind: "private_action" / "fee_fallback",
    nullifier: lez_node.nullifier,
    commitment: lez_node.commitment,
    position: lez_node.position,
    encrypted_post_state: lez_node.encrypted_note,
} / {
    kind: "sealed_message",
    commitment: lez_node.commitment,
    position: lez_node.position,
    note: lez_node.encrypted_note,
} / {
    kind: "recovery_binding",
    address: lez_node.account_id,
    note: lez_node.encrypted_note,
}
```

Returns, block by block in ascending order, every encrypted note the chain
carries with the public values needed to decrypt and check it.

```cddl
; readiness: New
lez_node.subscribe_to_private_notes_request = {
    subscription: lez_node.subscription_id,
    ? from: lez_node.resume_from,
}

lez_node.subscribe_to_private_notes_response = {
    ended: lez_node.subscription_end,
} / {
    mismatch: lez_node.resume_mismatch,
} / {
    error: lez_node.error,
}

; readiness: New
lez_node.private_notes_event = {
    subscription: lez_node.subscription_id,
    notes: lez_node.note_block,
}
```

Delivers the same walk as new blocks arrive.

31. <a id="f-31"></a>The API MUST expose a walk, ascending by block from a given
    block, returning for each privacy-preserving transaction its hash, its index
    in the block, and whether its action applied, and for each encrypted note it
    carries the values a scanner needs to open and check it: for a private
    action, including the fee fallback's, its nullifier, its commitment with the
    commitment's position in the commitment set, and its encrypted post-state;
    for a sealed async message, its commitment, note, and position; and for a
    recovery binding, its address and note. Proofs, witness sets, and public
    bodies MUST be omitted; a clear publication, whose envelope is plaintext, is
    read from the publication listing, [#27](#f-27). The notes are in the stored
    transaction (`lez/indexer/service/protocol/src/lib.rs:362-441`), whether an
    action applied is in the block's receipts, and the positions are in the
    state's commitment set and publications, so the work is a compact projection
    of data the node holds. **[New]**
32. <a id="f-32"></a>The walk MUST cover every encrypted note whatever kind of
    private account it addresses: a regular private account, a private PDA, and
    a shared private account keyed from a group master secret, of either kind.
    The node needs no knowledge of which kind a note belongs to, and the walk
    MUST NOT depend on it. **[New]**
33. <a id="f-33"></a>The walk MUST NOT take a viewing key, account identifier,
    or other value tied to the consumer's accounts. Every caller asking for the
    same range MUST receive the same result, so the walk discloses to the node
    the range read and nothing else. **[New]**
34. <a id="f-34"></a>The walk MUST resume from a block identifier, with or
    without its hash, on the same terms as the block subscription, [#61](#f-61),
    and MUST also be delivered as a subscription, so a viewing key consumer
    following the tip neither polls nor misses a block. **[New]**
35. <a id="f-35"></a>The API documentation MUST state what the walk serves for
    each kind of note, and that it serves every encrypted note in the range,
    whichever account it addresses. What a viewing key holder can and cannot
    establish from it belongs in the LEZ wallet API's documentation, since the
    wallet does the scanning. **[New]**

##### Commitment membership proofs

Reading the inclusion proofs a caller needs to spend a private account state or
consume a received message, and the root they are proven against. Zcash is the
one surveyed chain with a comparable structure. It serves the state of its
commitment tree at a height, `z_gettreestate` on the node and a tree-state read
on lightwalletd, from which a light client keeps its own witnesses up to date
rather than asking for a proof of the notes it holds
([Appendix: Blockchain API and SDK Ecosystem, section 1.29](../appendix/blockchain-api-sdk-ecosystem.md#129-read-historical-state-at-a-past-version)).

**Where the inputs come from.** A caller computes the commitment of each private
account it holds from the account's identifier and state
(`lee/state_machine/core/src/commitment.rs:56-59`), a state it learns from its
own transactions or from the viewing key walk ([#31](#f-31)). It learns the
position of each message addressed to it from the publication listing
([#27](#f-27)), or from the walk for a sealed message. The proofs answer at the
Bedrock status the caller chooses and name the block they were served at
([#97](#f-97)), so the caller knows which root they belong to. The single
position read the sequencer service also serves, `getMessagePath`
(`lez/sequencer/service/rpc/src/lib.rs:119-123`), is this call with one message
position and no commitment.

```cddl
; readiness: Ready, not exposed (#36, #37); New (#38)
lez_node.query_commitment_proofs_request = {
    commitments: [* lez_node.commitment],
    message_positions: [* lez_node.position],
    ? bedrock_status: lez_node.bedrock_status,
}

; one entry per requested item, in request order; `proof` is omitted for an
; item the set does not hold
lez_node.query_commitment_proofs_response = {
    root: lez_node.hash,
    commitment_proofs: [* lez_node.proof_entry],
    message_proofs: [* lez_node.proof_entry],
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.proof_entry = {
    ? proof: lez_node.membership_proof,
}

lez_node.membership_proof = {
    position: lez_node.position,
    siblings: [* lez_node.hash],
}
```

Returns a membership proof per requested commitment and per requested message
position, together with the commitment set root they are proven against.

36. <a id="f-36"></a>The API MUST return, for a list of commitments and a list
    of message positions, a membership proof per entry in request order and the
    commitment set root those proofs are proven against. A commitment or
    position the set does not hold MUST be reported as absent per entry rather
    than failing the call. This is the data the sequencer service's
    `getProofsAndRoot` returns today
    (`lez/sequencer/service/rpc/src/lib.rs:131-145`), which the wallet reaches
    the sequencer for (`lez/wallet/src/lib.rs:1310-1325`), and the state the
    node already holds answers it (`lee/state_machine/src/state/mod.rs:206`,
    `lee/state_machine/src/state/mod.rs:431-438`). **[Ready, not exposed]**
37. <a id="f-37"></a>The proofs and the root returned by one call MUST be
    consistent with one another: every proof verifies against the returned root,
    as those `getProofsAndRoot` returns do. **[Ready, not exposed]**
38. <a id="f-38"></a>The API documentation MUST state that this read names to
    the node the commitments and message positions the caller is about to spend,
    which is what a node shared by several wallets learns from it, and Privacy
    [#7](#priv-7)'s list of disclosing calls MUST include it. Soft [#6](#soft-6)
    asks for a way to spend that names none. **[New]**

##### Transaction status

Answering how far a transaction has progressed and whether it succeeded
([Appendix: Blockchain API and SDK Ecosystem, section 1.24](../appendix/blockchain-api-sdk-ecosystem.md#124-get-transaction-status)).

- **Ethereum**: `eth_getTransactionReceipt`, whose absence means not yet mined.
- **Solana**: `getSignatureStatuses`, carrying an explicit commitment level.
- **Bitcoin**: `gettransaction` reports confirmations but is wallet scoped.
- **Zcash**: `gettransaction` for confirmations, wallet scoped as on Bitcoin.
  ZIP 315 specifies the confirmation policy, three confirmations for trusted
  funds and ten for untrusted, as wallet behaviour rather than as a node call.

```cddl
; readiness: Ready, not exposed (#40, #41); Computed, not persisted (#41); New (#39, #40, #42)
lez_node.query_transaction_status_request = {
    transaction_hash: lez_node.hash,
}

lez_node.query_transaction_status_response = {
    status: lez_node.transaction_status,
    ? location: lez_node.transaction_location,
    ? assessed_at: lez_node.l1_block,
    ? execution: lez_node.execution_outcome,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

lez_node.transaction_status =
    "not_known" /
    "received" /
    "no_longer_observed" /
    "pending" /
    "finalized"

lez_node.transaction_location = {
    block: lez_node.block_ref,
    index: uint32,
}

lez_node.l1_block = {
    slot: uint64,
    hash: lez_node.hash,
}

lez_node.execution_outcome = {
    applied: true,
} / {
    applied: false,
    reason: lez_node.revert_reason,
}

lez_node.revert_reason = {
    code: uint32,
    message: tstr .size (1..512),
}
```

Returns how far a transaction has progressed, where it sits, and whether its
execution succeeded.

39. <a id="f-39"></a>The API MUST expose a per-transaction status query
    returning a status from a documented set that distinguishes at minimum the
    following, and the documentation of each status MUST state what a caller may
    conclude from it, that documented meaning being what the API guarantees:

    - **not known**, on the terms of [#42](#f-42);
    - **received**: the transaction is in the mempool the node holds or observes
      ([#50](#f-50)), having been submitted through this node or seen on the LEZ
      peer-to-peer network;
    - **pending** (`BedrockStatus::Pending`): in a LEZ block carried by an L1
      block that is not yet final, which an L1 reorganisation or a competing
      inscription can still replace, returning the transaction to received;
    - **finalized** (`BedrockStatus::Finalized`): in a LEZ block carried by an
      L1 block that is final, so a caller acting on it is acting on a settled
      result. This is not a statement about how much of the chain the node has
      processed.

    `pending` and `finalized` are the LEZ `BedrockStatus` values of the same
    names (`lez/common/src/block.rs:118-122`), and mean what they mean for a
    read ([#97](#f-97)): a `pending` transaction is in a block a `pending` read
    sees and a `finalized` read does not yet see, and a `finalized` transaction
    is in a block both see. `Safe` has no transaction status, for the reason
    [#97](#f-97) gives. Before a transaction is `finalized` its status can move
    back as well as forward, and the API documentation MUST state when. The node
    reads blocks only once final (`lez/chain_state/src/zone_indexer.rs:54-86`)
    and stamps every block it stores `Finalized`
    (`lez/indexer/core/src/block_store.rs:362`); the sequencer service already
    follows inscriptions at the L1 tip and folds them into the blocks it serves
    at `pending` (`lez/chain_state/src/chain.rs:1-3`). No status covers a block
    before its inscription, since a node learns of a block only from its
    inscription
    ([Assumption A7](#a7-lez-blocks-are-not-propagated-over-the-lez-p2p-protocol)).
    **[New]**

40. <a id="f-40"></a>The per-transaction status query MUST return the identifier
    and hash of the block containing the transaction and the transaction's index
    in it, which the node's store already resolves from the transaction hash
    (`lez/indexer/core/src/block_store.rs:161-163`) **[Ready, not exposed]**,
    and the L1 block the status was assessed against, so a caller can tell how
    current the answer is. The store already records the L1 slot of the
    validated tip (`lez/indexer/core/src/block_store.rs:211-216`) **\[Ready, not
    exposed\]**; the rest is **[New]**.

41. <a id="f-41"></a>The per-transaction status query MUST report execution
    success or failure, with a typed reason on failure, for any transaction the
    node has executed. A charged transaction whose action fails is included in
    its block and keeps its fee, and its receipt records the failure
    (`lez/common/src/block.rs:79-84`), which the FFI block type and
    `query_transaction_receipt` carry; that outcome is **\[Ready, not
    exposed\]**: the block keeps a list of reverted transactions that the FFI
    block type drops (exported through receipts on `dev` since `f99ba435f`,
    after `v0.4.0-rc3`). The reason is computed as free text when the action
    fails and then discarded
    (`lee/state_machine/src/validated_state_diff/mod.rs:66-72`), so a typed
    reason is **[Computed, not persisted]**.

42. <a id="f-42"></a>The API documentation MUST state that a transaction this
    node never saw, absent from the chain and from the mempool, is reported as
    not known, whether it was never submitted or was submitted elsewhere and
    discarded, no record of a declined transaction being retained. **[New]**

```cddl
; readiness: New
lez_node.query_status_request = {}

lez_node.query_status_response = {
    sync: lez_node.sync_state,
    tips: lez_node.tips,
    ? last_error: tstr .size (1..512),
    ? stall_reason: tstr .size (1..512),
    ? halt_verdict: tstr .size (1..512),
    peer_zones: [* lez_node.peer_zone],
    mode: lez_node.node_mode,
} / {
    error: lez_node.error,
}

lez_node.sync_state =
    "starting" /
    "syncing" /
    "caught_up" /
    "error" /
    "stalled" /
    "halted"

lez_node.peer_zone = {
    zone: lez_node.zone_id,
    health: "live" / "lagging" / "holed" / "suspended" / "pin_mismatch" / "halted",
}

lez_node.node_mode = {
    sequencer: true,
    accepting: bool,
} / {
    sequencer: false,
    relay_available: bool,
}
```

Returns the node's own state as a typed record: how far it has followed the
chain, its mode, and why it stopped if it has.

43. <a id="f-43"></a>The query MUST return a typed record, defined in the module
    contract like every other return type ([#96](#f-96)), carrying: the sync
    state, one of `starting`, `syncing`, `caught_up`, `error`, `stalled`, or
    `halted`; the `pending` and `finalized` tips, each as a block identifier and
    hash; the last error, the stall reason, and the cross-zone halt verdict when
    the node stopped; and the health of each peer zone it reads, one of `live`,
    `lagging`, `holed`, `suspended`, `pin_mismatch`, or `halted`. The node holds
    this data today (`lez/indexer/core/src/status.rs:16-24`, `:45-98`,
    `:159-161`), but the FFI's `query_status` returns it as a JSON string,
    untyped by any contract, and as a bare null pointer on any failure
    (`lez/indexer/ffi/src/api/query.rs:120-145`). **[New]**
44. <a id="f-44"></a>The status query MUST report the node's mode: whether it is
    in sequencer mode and, if it is, whether it is accepting transactions or
    still starting, as the sequencer service's `getSequencerStatus` reports
    today (`lez/sequencer/service/rpc/src/lib.rs:158-159`); and, outside
    sequencer mode, whether the node has a peer to relay a submission through. A
    caller learns before submitting whether its transaction is admitted here or
    relayed, and whether it can be either. **[New]**

##### Transaction submission

Handing a public or privacy-preserving transaction to the network
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

```cddl
; readiness: Ready, not exposed (#45); New (#45 to #48)
lez_node.submit_transaction_request = {
    transaction: lez_node.transaction,
}

lez_node.submit_transaction_response = {
    transaction_hash: lez_node.hash,
    accepted: "admitted" / "relayed",
} / {
    declined: lez_node.declined_reason,
    message: tstr .size (1..512),
} / {
    unavailable: "starting" / "no_relay_peer" / "mempool_full",
} / {
    error: lez_node.error,
}

lez_node.declined_reason =
    "malformed" /
    "stateless_check_failed" /
    "too_large" /
    "gas_above_block_limit" /
    "fee_below_price" /
    "payer_cannot_fund" /
    "payer_not_authorised" /
    "program_not_callable"
```

Accepts a public or privacy-preserving transaction and returns the hash it will
be known by.

45. <a id="f-45"></a>The API MUST accept a transaction of either kind, a public
    transaction signed by its signers or a privacy-preserving transaction
    carrying its proof (`lez/common/src/transaction.rs:20-23`), and returns the
    transaction hash a caller then tracks through `query_transaction_status`. A
    node in sequencer mode MUST admit the transaction to its own mempool, as the
    sequencer service does today through its RPC's `sendTransaction`
    (`lez/sequencer/service/rpc/src/lib.rs:52-53`) and its FFI's
    `sequencer_ffi_send_transaction`
    (`lez/core_ffi/src/sequencer/api/query.rs:408-438`), both taking either
    kind. **[Ready, not exposed]** Any other node MUST relay it over the LEZ
    peer-to-peer network to the nodes that sequence, which is not part of this
    API; no node outside sequencer mode submits today. **[New]**
46. <a id="f-46"></a>Every node MUST validate a transaction before it admits or
    relays it, against its state at `pending`, as the sequencer service screens
    fees against that state today
    (`lez/sequencer/actors/executor/src/actor.rs:369-384`), and a transaction it
    declines MUST be reported as a typed failure. The failures MUST distinguish
    at least: a malformed transaction; one that fails a stateless check; one too
    large for a block; one declaring more gas than a block allows; one whose
    fee, or whose proof-committed reserve less its tip, is below the current
    price at `pending`; one whose payer cannot fund the fee or is not authorised
    to pay it (`lez/sequencer/core/src/fees.rs:13-33`,
    `lez/programs/fee/core/src/validity.rs:17-39`); and one invoking a program a
    user transaction may not call. These MUST be reported separately from a node
    that cannot take a submission at all: one still starting, or, outside
    sequencer mode, one with no peer to relay through. A node in sequencer mode
    MUST also report a mempool that is full. The sequencer service's RPC gives
    three outcome codes of their own today, a full mempool, a privacy-preserving
    reserve below the current fee, and not yet online
    (`lez/sequencer/service/rpc/src/lib.rs:18-27`), and reports a public fee
    shortfall, an oversized transaction, and a call to a sequencer-only program
    as invalid parameters with text
    (`lez/sequencer/actors/rpc_server/src/actor/service.rs:58-68`,
    `lez/sequencer/actors/rpc_server/src/actor/service.rs:467-475`); the
    sequencer FFI reports every admission failure alike, apart from a
    transaction it cannot decode. A caller re-prices on an insufficient fee,
    funds the payer when it cannot pay, retries on a full mempool or a node that
    cannot take a submission yet, and does not retry the others. **[New]**
47. <a id="f-47"></a>Outside sequencer mode, acceptance means the node validated
    the transaction and handed it to the peer-to-peer network, not that a
    sequencer admitted it. Gossip forwards a transaction whatever a receiving
    sequencer's admission decides
    (`lez/sequencer/actors/gossip/src/actor.rs:528-540`), and a publication that
    cannot go out at once waits in a bounded queue that drops its oldest entry
    on overflow (`lez/sequencer/actors/gossip/src/actor.rs:66`,
    `lez/sequencer/actors/gossip/src/actor.rs:114`), so a sequencer's refusal
    never comes back to the relaying node, and acceptance may mean queued for
    relay. The API documentation MUST state both, and the caller learns what
    became of the transaction from the status query ([#39](#f-39)). **[New]**
48. <a id="f-48"></a>Submission MUST NOT sign or modify the transaction it is
    given. What is relayed MUST be the transaction accepted, re-encoded at most,
    and the hash returned MUST be the hash of that transaction. **[New]**

##### Mempool

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

A privacy-preserving transaction's proof commits to its private actions, its
recovery bindings, and the messages crossing between its private and public
execution
(`lee/state_machine/src/privacy_preserving_transaction/circuit/mod.rs:77-79`,
`lee/state_machine/core/src/circuit_io.rs:506-523`), while its public part runs
against the state at block production
(`lee/state_machine/src/validated_state_diff/mod.rs:380-408`). A pending
transaction can still defeat one being built: one that binds the same private
address first has it dropped at settlement
(`lee/state_machine/src/state/mod.rs:535-551`), and one that changes a public
account whose replies to private actors the proof predicted has it included as
reverted, its fee charged
(`lee/state_machine/core/src/execution_state.rs:930-941`,
`lee/state_machine/src/validated_state_diff/mod.rs:426-470`). Proving takes tens
of seconds (`docs/benchmarks/cycle_bench.md:92-104`), so a wallet that checks
the mempool before proving and again before submitting avoids discarding a proof
or paying for a revert. A spent nullifier needs no such check, since only the
key holder can spend it, and a changed nonce costs a new signature rather than a
new proof.

The mempool also answers where a submitted transaction has got to. Absence from
the chain alone means nothing, since a transaction not yet included and one the
network declined look identical, and only four of the nine surveyed chains
expose a mempool to tell them apart. A charged transaction whose action fails
during block production is included as reverted, but one that fails settlement
is dropped without a record: it is logged and omitted from the block
(`lez/sequencer/core/src/lib.rs:1793-1804`), so no drop is retrievable
afterwards and the mempool is the only surface on which its disappearance is
visible. Detecting a conflict with a transaction being built, together with
status, is what makes the mempool a requirement here.

Which mempool a node can show depends on its mode. A sequencer's mempool is a
bounded mempool, fed both by submission and by gossip from its peers, and it is
only pushed to today (`lez/mempool/src/lib.rs:66-92`). A node outside sequencer
mode holds none, but the transactions sequencers gossip travel on a peer-to-peer
topic per zone (`lez/sequencer/actors/gossip/src/actor.rs:338-340`) that any
node following the zone can observe. Either way, one transaction sits in several
mempools at once, none of them observable today, until it appears in a block
([Appendix: Blockchain API and SDK Ecosystem, section 1.26](../appendix/blockchain-api-sdk-ecosystem.md#126-inspect-the-mempool-or-pending-set)).

```cddl
; readiness: New
lez_node.query_mempool_digests_request = {
    ? cursor: lez_node.cursor,
    limit: lez_node.limit,
}

lez_node.query_mempool_digests_response = {
    digests: [* lez_node.mempool_digest],
    next_cursor: lez_node.cursor,
    size: uint64,
    bound: uint64,
    expiry_seconds: uint64,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

; `account_ids` and `nonces` name public accounts; `nullifiers` are those it
; spends, and `bound_addresses` the private addresses it binds
lez_node.mempool_digest = {
    transaction_hash: lez_node.hash,
    account_ids: [* lez_node.account_id],
    nonces: [* lez_node.account_nonce],
    nullifiers: [* lez_node.nullifier],
    bound_addresses: [* lez_node.account_id],
}

lez_node.account_nonce = {
    account_id: lez_node.account_id,
    nonce: lez_node.nonce,
}
```

Lists the mempool as digests: per entry, its transaction hash, the public
account identifiers and nonces it touches, the nullifiers it spends, and the
private addresses it binds.

49. <a id="f-49"></a>The API MUST list mempool transactions as digests carrying,
    per entry, its transaction hash, the public account identifiers and nonces
    it touches, the nullifiers it spends, and the private addresses it binds, so
    a caller detects a conflict with a transaction it is about to build without
    fetching bodies. **[New]**
50. <a id="f-50"></a>In sequencer mode the listing MUST be the mempool the node
    builds blocks from. Outside sequencer mode, the listing MUST be the
    transactions the node has observed on the LEZ peer-to-peer network and not
    yet seen inscribed, together with those submitted through it. In either mode
    an entry another node has included MUST leave the listing once the
    inscription is seen. The API documentation MUST state which a node serves,
    and that in neither mode does absence from the listing show that a
    transaction is not pending elsewhere. **[New]**

**Open problem: blocks produced but not yet inscribed.** A node in sequencer
mode holds the blocks it has produced and not yet seen inscribed on the Logos
Blockchain, whose transactions have left its mempool
(`lez/sequencer/core/src/lib.rs:1587-1591`), and since LEZ blocks are not
propagated over the LEZ p2p protocol
([Assumption A7](#a7-lez-blocks-are-not-propagated-over-the-lez-p2p-protocol)),
no other node knows of them. What the API reports about them is open: whether
the mempool listing, the status query, or `pending` reads on the node that
produced them show them, and how. Until that is settled, a transaction in such a
block is neither in the producing node's mempool nor in a block a `pending` read
sees, so no status of [#39](#f-39) describes it there, and a caller's conflict
check against the mempool can miss it.

51. <a id="f-51"></a>Outside sequencer mode, the observed set MUST admit only a
    transaction that passes the node's validation ([#46](#f-46)), as the gossip
    actor evaluates each transaction it receives
    (`lez/sequencer/actors/gossip/src/actor/validation.rs:57-89`), and is
    bounded and expires its entries as [#52](#f-52) requires. Once an entry has
    expired, the status query MUST report it as no longer observed rather than
    as received. **[New]**

52. <a id="f-52"></a>The mempool a node holds MUST be bounded by an operator
    setting, with a documented default, that limits its size, and the
    documentation MUST state the setting's unit. An entry MUST leave the mempool
    after a time set by a second operator setting, also with a documented
    default. This holds in either mode, for a sequencer's mempool and for the
    set a node outside sequencer mode observes on gossip alike. The sequencer's
    `mempool_max_size` (`lez/sequencer/core/src/config.rs:72-73`) is the prior
    art: a transaction count with no default in code, which the shipped devnet
    and testnet configurations set to 10000
    (`lez/sequencer/service/networks/devnet.json:4`). Bitcoin Core bounds its
    mempool with `-maxmempool` and expires its entries with `-mempoolexpiry`,
    with defaults of 300 MB and 336 hours. **[New]**

53. <a id="f-53"></a>The listing MUST be bounded and paginated by a cursor that
    is stable as [#68](#f-68) requires, a caller resuming from one neither
    skipping nor repeating an entry still in the mempool. The mempool turns over
    as entries are included or dropped, so an offset into it names a different
    entry from one call to the next. **[New]**

54. <a id="f-54"></a>A page of the digest listing MUST cost no more to serve
    than a page of blocks of the same limit, measured under Performance
    [#1](#perf-1), so a wallet re-checks for a conflict before proving and again
    before submitting. **[New]**

55. <a id="f-55"></a>The mempool MUST be queryable by transaction hash, through
    `query_mempool_entry` below, so a caller holding the hash
    `submit_transaction` returned asks whether that transaction is still pending
    without walking the listing. The listing MUST also report the mempool's size
    (`lez/mempool/src/lib.rs:30-32`), the bound it is configured with, and the
    time after which an entry leaves it ([#52](#f-52)), which a caller retrying
    on a full mempool reads. **[New]**

```cddl
; readiness: New
lez_node.query_mempool_entry_request = {
    transaction_hash: lez_node.hash,
}

lez_node.query_mempool_entry_response = {
    ? transaction: lez_node.transaction,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}
```

Returns one mempool transaction in full, by its transaction hash.

56. <a id="f-56"></a>The API MUST return a mempool transaction's body for a
    transaction hash, in the same shape `query_transaction` returns an included
    one, and MUST report absence distinctly from failure: an entry that has left
    the mempool since the caller last saw it is reported as absent rather than
    as an error. **[New]**

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

```cddl
; readiness: Ready, not exposed (#57); New (#58)
lez_node.subscribe_to_finalized_blocks_request = {
    subscription: lez_node.subscription_id,
    ? from: lez_node.resume_from,
}

lez_node.subscribe_to_finalized_blocks_response = {
    ended: lez_node.subscription_end,
} / {
    mismatch: lez_node.resume_mismatch,
} / {
    error: lez_node.error,
}

; readiness: Ready, not exposed (#57); New (#58)
lez_node.finalized_block_event = {
    subscription: lez_node.subscription_id,
    block: lez_node.block_ref,
}
```

Delivers each newly finalised block to the caller as an event, optionally
resuming from a position the consumer already processed.

57. <a id="f-57"></a>The API MUST expose a subscription to finalised blocks that
    notifies a registered consumer of each block as it becomes `finalized`,
    exporting the existing `subscribeToFinalizedBlocks` node RPC method. The
    consumer MUST be notified through `lez_node.finalized_block_event`
    deliveries rather than by polling, and reads a block it is told of through
    `query_block_by_hash`. **[Ready, not exposed]**
58. <a id="f-58"></a>Each notification MUST carry the block's height and its
    hash together, so a consumer knows which block occupied the position without
    a second read. The existing subscription yields a height alone
    (`lez/indexer/service/rpc/src/lib.rs:45`), which does not identify a block
    once a block a `pending` read sees can be replaced; the pair already exists
    as `BlockMeta` (`lez/common/src/block.rs:11-14`). **[New]**

```cddl
; readiness: New
lez_node.subscribe_to_pending_blocks_request = {
    subscription: lez_node.subscription_id,
    ? from: lez_node.resume_from,
}

lez_node.subscribe_to_pending_blocks_response = {
    ended: lez_node.subscription_end,
} / {
    mismatch: lez_node.resume_mismatch,
} / {
    error: lez_node.error,
}

; readiness: New
lez_node.pending_block_event = {
    subscription: lez_node.subscription_id,
    change: lez_node.pending_block_change,
}

lez_node.pending_block_change = {
    kind: "entered",
    block: lez_node.block_ref,
} / {
    kind: "replaced",
    replaced: lez_node.block_ref,
    by: lez_node.block_ref,
} / {
    kind: "finalized",
    block: lez_node.block_ref,
}
```

Delivers each change to the blocks a `pending` read sees to the caller as an
event: a block entering them on inscription, a block replaced, with the block
that replaces it, and a block becoming `finalized`, optionally resuming from a
position the consumer already processed.

59. <a id="f-59"></a>The API MUST expose a subscription to the blocks a
    `pending` read sees, notifying a consumer of each block as it enters them on
    inscription; of each such block that is replaced, with the block that
    replaces it; and of each such block as it becomes `finalized`. The sequencer
    service already folds each L1 update into the blocks it serves at `pending`
    as an extension or a conflict with the blocks it orphans
    (`lez/sequencer/core/src/lib.rs:350-358`). **[New]**
60. <a id="f-60"></a>A notification that cannot be delivered, because the
    consumer is not keeping up, MUST end the subscription with a final response
    carrying the last position delivered, `last_delivered`, so the consumer
    resumes from it ([#61](#f-61)) rather than missing a block. The block
    subscription logs a failed send and goes on to the next block today
    (`lez/indexer/service/src/service.rs:526-534`). A resume position below the
    retention floor MUST be answered as [#91](#f-91) defines. **[New]**
61. <a id="f-61"></a>The block subscription MUST accept a start position: the
    identifier of the block the consumer last processed, and optionally its
    hash. Delivery MUST resume from the block after that position, so a consumer
    that reconnects observes no gap. Given an identifier alone, delivery MUST
    resume after the block the node holds at that identifier. Given an
    identifier and a hash, the node MUST check the hash against the block it
    holds at that identifier. On a mismatch it MUST NOT resume, and it MUST
    report the mismatch together with the last block on its chain that the
    caller's position shares, so the caller rolls back to it in one round trip.
    Neither existing RPC subscription carries a position:
    `subscribeToFinalizedBlocks` takes no arguments, and `subscribeToEvents`
    filters by transaction, program, and selector but not by position
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.33](../appendix/blockchain-api-sdk-ecosystem.md#133-resume-a-stream-from-a-known-position)).
    **[New]**

Base-layer RPCs resume by height and leave reorg detection to the client:
Ethereum's `eth_getLogs` accepts a `blockHash` only instead of a block range,
never with one (EIP-234), and Solana relies on commitment levels. Reorg-aware
consumers each rebuild the check: The Graph keeps its position as a block number
and hash, Firehose cursors encode the block number and hash, and Cardano's chain
sync resumes from slot and hash points. Reads here default to `pending`, where a
block can be replaced
([Assumption A8](#a8-every-node-serves-pending-reads-as-well-as-finalized-ones-and-reads-default-to-pending))
and the walk is a range read, so the node performs the check once for every
caller. A caller that sends no hash resumes by height alone, as on those base
layers
([Appendix: Blockchain API and SDK Ecosystem, section 1.33](../appendix/blockchain-api-sdk-ecosystem.md#133-resume-a-stream-from-a-known-position)).

```cddl
; readiness: New
lez_node.unsubscribe_request = {
    subscription: lez_node.subscription_id,
}

lez_node.unsubscribe_response = {
    result: "cancelled" / "not_found",
} / {
    error: lez_node.error,
}
```

Cancels a registered subscription and releases the resources it holds.

62. <a id="f-62"></a>Cancelling a subscription MUST stop delivery, and an event
    for it MUST NOT be delivered after `unsubscribe` returns. **[New]**

```cddl
; readiness: Ready, not exposed (#63); New (#64)
lez_node.subscribe_to_events_request = {
    subscription: lez_node.subscription_id,
    filter: lez_node.event_filter,
    ? from: lez_node.resume_from,
}

lez_node.event_filter = {
    ? transaction_hash: lez_node.hash,
    ? program_account_id: lez_node.account_id,
    ? selector: lez_node.selector,
}

lez_node.subscribe_to_events_response = {
    ended: lez_node.subscription_end,
} / {
    mismatch: lez_node.resume_mismatch,
} / {
    error: lez_node.error,
}

; readiness: Ready, not exposed (#63); New (#64)
lez_node.event_record_event = {
    subscription: lez_node.subscription_id,
    record: lez_node.event_record,
}
```

Delivers each new event record matching a filter to the caller as an event,
optionally resuming from a position the consumer already processed.

63. <a id="f-63"></a>The API MUST expose a subscription to program events,
    exporting the existing `subscribeToEvents` node RPC method
    (`lez/indexer/service/rpc/src/lib.rs:48-49`) with the filter it takes by
    transaction, program, and selector, and delivering each record in the shape
    `query_events` returns. Cancellation, and delivery to the caller that opened
    the subscription alone, MUST be on the same terms as the block subscription.
    Today the subscription stops at the node RPC, so no application can follow a
    program's events. A filter the node's event filter does not persist MUST be
    refused at subscription, as the RPC refuses it
    (`lez/indexer/service/src/service.rs:80-87`), rather than accepted and left
    silent. **[Ready, not exposed]**
64. <a id="f-64"></a>The event subscription MUST resume from a block identifier,
    with or without its hash, on the same terms as the block subscription,
    [#61](#f-61), so a consumer following a program's events across a
    disconnection misses none. A resume position outside the node's event-filter
    history MUST be answered as data the node does not hold, as [#13](#f-13)
    defines for the query. **[New]**

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

```cddl
; readiness: New
lez_node.query_transactions_by_account_request = {
    account_id: lez_node.account_id,
    ? cursor: lez_node.cursor,
    limit: lez_node.limit,
    order: lez_node.order,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_transactions_by_account_response = {
    entries: [* lez_node.history_entry],
    next_cursor: lez_node.cursor,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.history_entry = {
    transaction: lez_node.transaction,
    location: lez_node.transaction_location,
    applied: bool,
}
```

Returns one page of the transactions touching an account, resumed from a cursor
the previous page returned. Exported today as a numeric offset into the
per-account index.

65. <a id="f-65"></a>The query MUST return at most `limit` transactions from the
    position the cursor names, or from the start of the walk when the cursor is
    absent, and MUST stop early at the end of the account's history rather than
    failing. The bounded read exists; the cursor replaces the offset it takes
    today. **[New]**
66. <a id="f-66"></a>Each entry MUST carry, beside the transaction, the
    identifier and hash of its block, its index in the block, and whether its
    action applied, so a deposit tracker judges each entry without a status call
    per entry. History entries are bare transactions today
    (`lez/indexer/ffi/src/api/types/transaction.rs:343-346`). **[New]**
67. <a id="f-67"></a>`query_transactions_by_account` MUST accept an ordering
    parameter supporting both oldest-first and newest-first. Newest-first is the
    order a deposit tracker reads in. **[New]**
68. <a id="f-68"></a>The page MUST be walked by a cursor rather than by a
    numeric offset, and the cursor MUST be stable: a caller resuming from one
    neither skips nor repeats an entry that existed when the walk began, however
    many transactions have landed or been pruned since. The per-account index is
    append-only (`lez/storage/src/indexer/write_atomic.rs:64-76`), so an offset
    from the oldest entry holds that property, but an offset from the newest,
    which is how a deposit tracker reads, shifts with every arrival, and so does
    any offset once history is pruned. What a cursor holds is the implementer's
    choice, and MAY differ from one paginated read to another. **[New]**

```cddl
; readiness: Ready (#69, #70); New (#70 to #72)
lez_node.query_blocks_request = {
    ? from: lez_node.block_id,
    limit: lez_node.limit,
    order: lez_node.order,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_blocks_response = {
    blocks: [* lez_node.block],
    ? next_from: lez_node.block_id,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns a page of blocks after `from`, `from` itself excluded, walked in the
requested direction. With `from` absent, a newest-first walk starts at the tip
at the requested Bedrock status and an oldest-first walk at the retention floor.
Exported today as `query_block_vec`, which descends only.

69. <a id="f-69"></a>The query MUST return at most `limit` blocks, descending
    from `from`, or from the tip at the requested Bedrock status ([#97](#f-97))
    when `from` is absent. **[Ready]**
70. <a id="f-70"></a>The API documentation MUST state that the bound is
    exclusive in both directions. The store already implements it that way when
    descending, from `before_id.saturating_sub(1)`
    (`lez/storage/src/indexer/read_multiple.rs:11-12`) **[Ready]**; ascending is
    new with [#71](#f-71) **[New]**.
71. <a id="f-71"></a>`query_blocks` MUST accept an ordering parameter supporting
    both oldest-first and newest-first, on the same terms as
    `query_transactions_by_account`. Oldest-first is the order a consumer
    scanning forward reads in: a wallet syncing private accounts walks ascending
    from the last block it processed to the tip, decrypting each
    privacy-preserving transaction body against its own viewing key. Descending
    from the tip cannot serve that walk. An ascending walk with `from` absent
    MUST start at the retention floor. **[New]**
72. <a id="f-72"></a>Every paginated response MUST carry the position a caller
    resumes from. A caller pages until it receives an empty page: a page shorter
    than `limit` does not mark the end of the result set, and the API
    documentation MUST say so. For the account history and the mempool listing
    that position is a stable cursor ([#68](#f-68)); for the walks keyed by
    position, the blocks, publications, and encrypted notes, it is the block or
    commitment-set position the next page starts after. Today the block walk and
    the account history return a bare list with no resume position, so a caller
    can infer the end only from a short or empty page. **[New]**

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

```cddl
; readiness: New
lez_node.query_version_request = {}

lez_node.query_version_response = {
    version: tstr .size (1..64),
    build: tstr .size (1..128),
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}
```

Returns the software the node is running, so a caller knows which build answered
it
([Appendix: Blockchain API and SDK Ecosystem, section 1.1](../appendix/blockchain-api-sdk-ecosystem.md#11-get-node-version-and-software-identity)).
Every surveyed chain reports this and LEZ reports none: `web3_clientVersion` on
Ethereum, `getVersion` on Solana, `getnetworkinfo` on Bitcoin and Zcash.

73. <a id="f-73"></a>The API MUST report the version of the node serving the
    call, together with enough build identity to tell two builds of the same
    version apart. An integrator that hits a defect can then say which build
    produced it, and one that knows a build is bad can route around it. Neither
    the FFI's `query_status` nor the node RPC's `checkHealth`, which the FFI
    does not export, reports a version
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**
74. <a id="f-74"></a>The version MUST change only across restarts, so a caller
    MAY cache it for the life of a connection and use it to detect that the
    surface it holds a schema for has been replaced. **[New]**

```cddl
; readiness: Ready (#75, #76); Ready, not exposed (#77); Computed, not persisted (#77)
lez_node.query_block_request = {
    block_id: lez_node.block_id,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_block_response = {
    ? block: lez_node.block,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.block = {
    header: lez_node.block_header,
    transactions: [* lez_node.block_transaction],
    bedrock_status: lez_node.bedrock_status,
}

lez_node.block_header = {
    id: lez_node.block_id,
    prev_hash: lez_node.hash,
    hash: lez_node.hash,
    timestamp: lez_node.timestamp,
}

; `receipt` is omitted for the fee and clock transactions that close a block
lez_node.block_transaction = {
    transaction: lez_node.transaction,
    ? receipt: lez_node.receipt,
}
```

Returns the block at a given identifier: its header, its full transaction body,
and its Bedrock status.

75. <a id="f-75"></a>The query MUST return the stored block for a known
    identifier and an omitted `block` for one the node does not hold, keeping
    the two distinct from a backend failure. **[Ready]**
76. <a id="f-76"></a>The returned header MUST carry the block identifier,
    previous block hash, own hash, and timestamp
    (`lez/indexer/service/protocol/src/lib.rs:178-184`), and the body MUST carry
    every transaction in the block. **[Ready]**
77. <a id="f-77"></a>The returned body MUST carry each user transaction's
    receipt, so a consumer reading a block can tell a transaction that took
    effect from one that only paid its fee. The stored block, the node RPC, and
    the FFI block type already carry them
    (`lez/indexer/service/protocol/src/lib.rs:207-252`,
    `lez/indexer/ffi/src/api/types/block.rs:10-45`). Whether an action applied
    is **[Ready, not exposed]** and its gas and fee are **\[Computed, not
    persisted\]** (all carried in receipts on `dev` since `f99ba435f`, after
    `v0.4.0-rc3`).

```cddl
; readiness: Ready
lez_node.query_block_by_hash_request = {
    hash: lez_node.hash,
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_block_by_hash_response = {
    ? block: lez_node.block,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}
```

Returns the same block record, resolved by block hash rather than by height. The
store already holds the hash as a secondary index onto the height
(`lez/storage/src/indexer/read_once.rs:67-70`).

78. <a id="f-78"></a>The query MUST return the same record `query_block` returns
    for the corresponding height, and an omitted `block` for a hash no indexed
    block carries. **[Ready]**

```cddl
; readiness: Ready
lez_node.query_last_block_request = {}

; `block_id` is omitted while no block is `finalized`
lez_node.query_last_block_response = {
    ? block_id: lez_node.block_id,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}
```

Returns the identifier of the node's last finalised block.

79. <a id="f-79"></a>The query MUST return the last finalised block identifier,
    and MUST report an empty chain as an outcome distinct from an error.
    **[Ready]**

```cddl
; readiness: New
lez_node.query_chain_tip_request = {}

lez_node.query_chain_tip_response = {
    ? pending: lez_node.tip,
    ? finalized: lez_node.tip,
} / {
    error: lez_node.error,
}

lez_node.tip = {
    block: lez_node.block_ref,
    timestamp: lez_node.timestamp,
}
```

Returns the tip as one record rather than as a height the caller must then
resolve.

80. <a id="f-80"></a>The API MUST expose the chain tip as a single call
    returning, at `pending` and at `finalized`, the tip block's height together
    with its hash and timestamp, so learning about the tip does not cost a
    second call and a caller knows which block holds the position. Below the
    FFI, `getLastFinalizedBlockId` returns a height alone; the chain state keeps
    a tip as a height and hash (`lez/chain_state/src/apply.rs:38-41`), and the
    timestamp comes from a second read. **[New]**

```cddl
; readiness: Ready, not exposed (#81); New (#82 to #84)
lez_node.query_network_identity_request = {}

lez_node.query_network_identity_response = {
    zone: lez_node.zone_id,
    chain: lez_node.chain_identity,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

lez_node.chain_identity = {
    id: tstr .size (1..256),
} / {
    unavailable: true,
}
```

Returns what the node is following: which zone, and which Logos Blockchain chain
that zone settles to.

81. <a id="f-81"></a>The call MUST return the zone identifier the node is
    following, so an application can confirm which zone it is connected to. The
    node's channel configuration already holds it
    (`lez/indexer/core/src/config.rs:37`). **[Ready, not exposed]**
82. <a id="f-82"></a>The same call MUST return the chain identifier of the Logos
    Blockchain the zone settles to. A zone identifier alone does not distinguish
    the same zone running against different L1 networks, which is the case an
    integrator connecting to the wrong network hits first, and the two are
    answered together so a caller cannot check one and assume the other.
    **[New]**
83. <a id="f-83"></a>The chain identifier MUST be the one inscribed in the L1
    genesis block as a Cryptarchia parameter, a bounded UTF-8 string such as
    `logos-chain-1`, and not a value the node is configured with independently:
    a configured string would agree with whatever an operator typed rather than
    with the chain the node is settling to. The Logos Blockchain node serves it,
    read off the genesis inscription, at `GET /chain/id` (`logos-blockchain`
    `nodes/api-common/src/paths.rs:11`), and the node already builds a client
    for its L1 node (`lez/indexer/core/src/lib.rs:112-115`), so the node reads
    it from there. **[New]**
84. <a id="f-84"></a>Where the chain identifier cannot be obtained, because the
    L1 node is unreachable, the call MUST report it as unavailable rather than
    omitting it or returning a placeholder, so a caller can tell an unidentified
    chain from an unasked question. **[New]**

```cddl
; readiness: New
lez_node.query_chain_parameters_request = {
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_chain_parameters_response = {
    parameters: lez_node.chain_parameters,
    chain: lez_node.chain_identity,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.chain_parameters = {
    max_transaction_bytes: uint64,
    gas_caps: lez_node.gas,
    proof_verification_gas: uint64,
    proof_charged_bytes: uint64,
    min_private_refund: lez_node.amount,
    base_fee_change: lez_node.ratio,
}

lez_node.ratio = {
    numerator: uint32,
    denominator: uint32,
}
```

Returns the protocol parameters a transaction is built against, with the block
they were read at.

85. <a id="f-85"></a>The API MUST return the chain parameters a caller needs
    before it builds and proves a transaction, at the requested Bedrock status
    ([#97](#f-97)): the largest transaction a block accepts, which is the
    channel's maximum block size
    (`lez/programs/sequencer_stake/core/src/lib.rs:238-252`,
    `lez/sequencer/core/src/lib.rs:2490`); the per-block execution and storage
    gas caps; the fixed execution gas a privacy-preserving transaction is
    charged for verification and the length its proof is charged at; the
    smallest surplus refunded rather than kept as tip; and the rate at which a
    base fee moves between blocks (`lez/programs/fee/core/src/market.rs:15-54`).
    A wallet that learns a transaction is too large only on submitting it has
    already spent tens of seconds proving it. The channel parameters change by
    committee vote, so each value MUST be reported with the block it was read
    at, and the chain identifier of [#82](#f-82) MUST be returned with them.
    **[New]**

```cddl
; readiness: Ready, not exposed (#87); New (#86)
lez_node.query_program_ids_request = {
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_program_ids_response = {
    programs: [* lez_node.program_entry],
    circuit_id: lez_node.hash,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.program_entry = {
    account_id: lez_node.account_id,
    image_id: lez_node.hash,
    ? builtin_name: tstr .size (1..64),
}
```

Returns the programs deployed on the chain, each with its account identifier,
its image identifier, and, for a builtin, its name, together with the identifier
of the privacy-preserving circuit the chain verifies proofs against.

86. <a id="f-86"></a>The API MUST expose the programs deployed on the chain as
    its state records them, programs deployed after genesis included. A program
    is an account written through the program loader, and the state resolves an
    account to the image deployed at it
    (`lee/state_machine/src/state/mod.rs:408-428`). An integrator decoding
    account data needs this: each actor state is keyed by the account identifier
    of its program, and decoding it means knowing which program that is. The
    state resolves one account to its program but keeps no index of the programs
    deployed, so listing them is an index built on ingest or a scan, and builtin
    names come from the builtin program list
    (`lez/programs/src/lib.rs:152-172`). The sequencer service's `getProgramIds`
    returns a fixed list today
    (`lez/sequencer/actors/rpc_server/src/actor/service.rs:275-283`). **[New]**
87. <a id="f-87"></a>The same call MUST return the identifier of the
    privacy-preserving circuit whose proofs this node verifies, which a wallet
    compares with its own before it proves: a mismatch otherwise surfaces only
    when a transaction that took tens of seconds to prove is rejected. That
    identifier is the one entry `getProgramIds` returns today, and the wallet's
    health check compares it with its own (`lez/wallet/src/cli/mod.rs:300-309`).
    **[Ready, not exposed]**

```cddl
; readiness: Ready, not exposed
lez_node.query_fee_state_request = {
    ? bedrock_status: lez_node.bedrock_status,
}

lez_node.query_fee_state_response = {
    base_fees: lez_node.gas_prices,
    next_block_lowest: lez_node.gas_prices,
    next_block_highest: lez_node.gas_prices,
    gas_caps: lez_node.gas,
    served: lez_node.served_at,
} / {
    error: lez_node.error,
}

lez_node.gas_prices = {
    execution: lez_node.amount,
    storage: lez_node.amount,
}
```

Returns the fee market a transaction is priced against: the base fees at a
stated block, the band the next block's may move within, and the per-block gas
caps
([Appendix: Blockchain API and SDK Ecosystem, section 1.17](../appendix/blockchain-api-sdk-ecosystem.md#117-estimate-the-fee-or-fee-rate)).

88. <a id="f-88"></a>The API MUST expose the execution and storage base fees,
    the block height they were read at, and the per-block execution and storage
    gas caps, from the fee state the chain state holds
    (`lez/chain_state/src/apply.rs:335-344`). The wallet prices every fee
    declaration and every in-proof reserve against these values, read today from
    the sequencer service's `getFeeState`
    (`lez/sequencer/service/rpc/src/lib.rs:57-58`), and the node FFI serves none
    of them. **[Ready, not exposed]**
89. <a id="f-89"></a>The same call MUST return the lowest and highest base fees
    the next block can open at, in each dimension, as `getFeeState` does, so a
    caller sets a cap on the fee from the band rather than from a guess. The
    band follows from the fee state and the fee market's constants
    (`lez/sequencer/core/src/fees.rs:89-123`). The API documentation MUST state
    that the band bounds the next block and is not an estimate for a given
    transaction. **[Ready, not exposed]**

The fee state is read at the Bedrock status the caller chooses ([#97](#f-97)),
and a wallet pricing a transaction reads it at `pending`, as the sequencer
service quotes it today. A base fee can move by up to an eighth between blocks
(`lez/programs/fee/core/src/market.rs:16`), so a band read at `finalized`, whose
tip on testnet trails the `pending` tip by about an hour, bounds only the block
after the `finalized` tip, and the API says so.

No tip history is required. A sequencer takes transactions from its mempool in
arrival order (`lez/mempool/src/lib.rs:40-58`,
`lez/sequencer/core/src/lib.rs:1591`), so at the commit read the tip, paid to
the block's producer whichever way a transaction pays, does not buy earlier
inclusion. Each receipt records the tip a transaction paid ([#77](#f-77)). For a
proof-committed fee that figure includes any surplus kept rather than refunded,
so it can exceed the tip the sender chose, which the transaction body carries.
Soft [#5](#soft-5) covers the case where inclusion comes to depend on the tip.

```cddl
; readiness: New
lez_node.query_retention_floor_request = {}

; `scopes` lists any data kept further back than the two floors
lez_node.query_retention_floor_response = {
    account_state: lez_node.block_id,
    transactions: lez_node.block_id,
    scopes: [* lez_node.retention_scope],
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

lez_node.retention_scope = {
    scope: tstr .size (1..64),
    floor: lez_node.block_id,
}
```

Returns the earliest block the node can still answer for, separately for account
state pinned to a block and for transaction retrieval.

90. <a id="f-90"></a>The API MUST expose the node's retention floor: the
    earliest block for which account state can be read at a pinned block
    identifier, and the earliest block for which a transaction can be retrieved.
    A deployment that retains everything reports genesis and one bounded by
    Reliability [#7](#rel-7) reports a moving value, and a caller learns which
    by asking rather than by discovering it through a failed read. Where a
    deployment keeps some data further back than the rest, as Soft [#4](#soft-4)
    allows, the call MUST report retention per scope rather than one floor that
    is wrong for part of the data. **[New]**
91. <a id="f-91"></a>A read for data below the retention floor MUST be answered
    as such, distinctly from a read for data that never existed. Asking for a
    block the node discarded and asking for one the chain never held are
    different questions with different answers: the first is retried against an
    archival source and the second is not. A caller learns which without a
    second call to the retention floor, which may itself have moved between the
    two. **[New]**

##### The node on the peer-to-peer network

What a caller can learn about the node's place in the LEZ peer-to-peer network.
The protocol between nodes is not part of this API
([Out of Scope](#out-of-scope)), but a node's connectivity is: a node with no
peers relays nothing and observes no mempool transaction.

- **Ethereum**: `net_peerCount` and `net_listening` for any caller; Geth's
  `admin_peers` and `admin_nodeInfo` for the operator.
- **Bitcoin**: `getconnectioncount`, `getpeerinfo`, and `getnetworkinfo`.
- **Cosmos**: CometBFT's `/net_info`.
- **Solana**: `getClusterNodes`.

```cddl
; readiness: New
lez_node.query_peer_count_request = {}

lez_node.query_peer_count_response = {
    inbound: uint32,
    outbound: uint32,
    sequencers: uint32,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}
```

Returns how many LEZ peers the node is connected to.

92. <a id="f-92"></a>The API MUST report how many LEZ peers the node is
    connected to, inbound and outbound, and how many of them are in sequencer
    mode where a peer makes that known, so a caller can tell a node that can
    relay and observe from one that cannot. It is in the user category. The
    sequencer's gossip actor already tracks the peers it is connected to
    (`lez/sequencer/actors/gossip/src/actor.rs:82-86`). **[New]**

```cddl
; readiness: New
lez_node.query_peers_request = {}

lez_node.query_peers_response = {
    peer_id: tstr .size (1..128),
    listen_addresses: [* tstr .size (1..256)],
    peers: [* lez_node.peer],
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

lez_node.peer = {
    peer_id: tstr .size (1..128),
    address: tstr .size (1..256),
    direction: "inbound" / "outbound",
    connected_at: lez_node.timestamp,
    ? sequencer: bool,
}
```

Returns the node's own peer identity and listening addresses, and each connected
peer.

93. <a id="f-93"></a>The API MUST return the node's peer identifier and
    listening addresses, and, for each connected peer, its identifier, address,
    direction, the time it connected, and, where known, whether it is in
    sequencer mode. A peer's address is another operator's network identity, so
    the function is in the operations category ([#104](#f-104)), as Geth keeps
    `admin_peers` off its public endpoints. **[New]**

##### Operating the node

What an operator watches to keep a node healthy and to answer a user who reports
a failed call
([Appendix: Blockchain API and SDK Ecosystem, section 2.4](../appendix/blockchain-api-sdk-ecosystem.md#24-per-call-logging-and-metrics)).

- **Geth**: request, success, and failure counts and serving-time histograms per
  method; each served call logged at debug level with its request identifier.
- **Bitcoin**: with `-debug=rpc`, each call logged with its method, the
  authenticated user, and its request identifier.

```cddl
; readiness: New
lez_node.query_metrics_request = {}

lez_node.query_metrics_response = {
    methods: [* lez_node.method_metrics],
    open_subscriptions: uint64,
    tips: lez_node.tips,
} / {
    error: lez_node.error,
}

lez_node.method_metrics = {
    method: tstr .size (1..128),
    calls: uint64,
    succeeded: uint64,
    failed: [* lez_node.failure_tally],
    serving_time: lez_node.histogram,
}

lez_node.failure_tally = {
    code: lez_node.error_code,
    calls: uint64,
}

; `buckets` holds one tally per bound, and one above the last
lez_node.histogram = {
    bounds_us: [* uint64],
    buckets: [* uint64],
}
```

Returns the node's call metrics, per method.

94. <a id="f-94"></a>The API MUST report, for each function of the user
    category, how many calls it served, how many succeeded and failed by outcome
    ([#101](#f-101), [#103](#f-103)), and the distribution of their serving
    times, together with the number of open subscriptions, as Geth reports per
    method. It is in the operations category ([#104](#f-104)). The node MUST NOT
    keep figures per caller: who called what is in Logos Core's call audit
    records. **[New]**
95. <a id="f-95"></a>The node MUST log each call, when the operator turns it on,
    with its method, its outcome, its serving time, the caller identity Logos
    Core attached to it, and the identifier Logos Core gives the call, so an
    operator can find a call a user reports. It MUST be off by default (Privacy
    [#5](#priv-5)), the operator chooses whether caller identities are recorded,
    and the documentation MUST state what each setting records. Bitcoin Core's
    `-debug=rpc` is the precedent. **[New]**

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

96. <a id="f-96"></a>The LEZ node module MUST publish its contract as a CDDL
    schema in the profile of
    [Assumption A9](#a9-the-api-defined-here-is-the-lez-node-modules-api),
    covering every method and event the module exposes: its request, its
    response, its error values, the category it belongs to ([#104](#f-104)), and
    the node modes it is available in ([#99](#f-99)). The profile carries no
    field for the category or the mode of a method, so the contract MUST express
    them another way the documentation states, such as separate user and
    administrative interfaces. The contract MUST be retrievable at run time, as
    the profile's `logos.schema` method serves it
    (`spec-module-interface.md:3222-3227`). The module wrapping the FFI today
    publishes a LIDL contract derived from the FFI header. The existing
    `getSchema` describes the block, event record, and event filter types rather
    than the surface, and is not exported
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**

Every function defined above is further bound by the following.

097. <a id="f-97"></a>The API MUST take a Bedrock status, `pending` or
     `finalized`, on every read of chain state, and MUST answer at it. A lookup
     by transaction hash, `query_transaction` and `query_transaction_receipt`,
     takes no status: it answers at `pending`, carries the tips alone, and
     leaves whether the transaction is final to `query_transaction_status`
     ([#39](#f-39)). The blocks the two statuses see are nested. A `finalized`
     read sees only the blocks inscribed in L1 blocks that are final. A
     `pending` read sees those, and also the blocks inscribed in L1 blocks that
     are not yet final. Only those latter blocks can be replaced, by an L1
     reorganisation or a competing inscription, and a caller tells them apart by
     the `finalized` tip every response carries ([#98](#f-98)). A caller that
     omits the status reads at `pending`, as the sequencer service answers today
     (`lez/sequencer/actors/executor/src/actor.rs:542-553`,
     `lez/sequencer/core/src/lib.rs:1956`). Each answer MUST name the status and
     the block, with its hash, it was served at, because an answer at `pending`
     can be overtaken when a block it saw is replaced. The two values are LEZ's
     `BedrockStatus` (`lez/common/src/block.rs:118-122`). `BedrockStatus` also
     declares `Safe`, but LEZ never assigns it: a block is created `Pending`,
     and is only ever set to `Finalized` or reset to `Pending`
     (`lez/common/src/block.rs:203`,
     `lez/sequencer/actors/storage/src/actor.rs:272-306`, `:1638`,
     `lez/indexer/core/src/block_store.rs:362`). Bedrock exposes only its tip
     and its last immutable block (`logos-blockchain`
     `c-bindings/src/api/cryptarchia.rs:63-70`), so the API omits `Safe` until a
     rule defines it. **[New]**
098. <a id="f-98"></a>The API MUST report, on every read, the tips it was served
     against: the identifier and hash of the last block visible at `pending` and
     at `finalized` when it answered. A caller detects a stalled or lagging node
     from any response, not only from the status query. No return type carries
     them today. **[New]**
099. <a id="f-99"></a>Every function MUST be available on every node unless the
     contract ([#96](#f-96)) marks it, or one of its outcomes, for sequencer
     mode. A node outside sequencer mode MUST answer such a call as not
     available in its mode, the `not_available_in_mode` error, an outcome
     distinct from not found, invalid argument, backend failure, and refusal
     ([#101](#f-101), [#108](#f-108)), so a caller learns to call a node in
     sequencer mode rather than retrying this one. **[New]**
100. <a id="f-100"></a>Every function above MUST be served by the node module
     under its contract ([#96](#f-96)) with the semantics defined here,
     including the not-found and error distinction required by Functionality
     [#101](#f-101). The module that wraps the FFI today carries eight of its
     queries, omitting `query_events`, and at `dev`, where the FFI has ten,
     `query_transaction_receipt` as well. **[New]**
101. <a id="f-101"></a>The API MUST signal not-found, invalid-argument, and
     backend failure as three distinguishable outcomes on every query: an
     omitted record or a `not_found` result, the `invalid_argument` error, and
     the `backend_failure` error. The module currently flattens not-found and
     failure into an empty string
     ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
     **[New]**
102. <a id="f-102"></a>Errors MUST be machine-readable and stable: a caller
     branches on a value rather than on message text, and that value MUST NOT
     change meaning between releases. Which values exist is for the
     implementation to decide; two failures a caller would recover from
     differently MUST NOT arrive as the same value. The node RPC returns the
     stock JSON-RPC `InternalError` code with free text for most failures,
     typing only an invalid event range, and the FFI's status type has five
     values, so most failures look alike
     ([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).
     **[New]**
103. <a id="f-103"></a>The distinctions a caller acts on MUST include, at
     minimum: a request that was malformed, which the caller fixes; a request
     for data below the retention floor, which the caller sends elsewhere; a
     request the node could not serve because it is stalled or lagging, which
     the caller retries; and a backend failure, which the caller retries with
     backoff. A transient failure MUST be distinguishable from a permanent one
     without parsing text. **[New]**

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

```cddl
lez_node.start_node_request = {
    config_path: tstr .size (1..4096),
    storage_dir: tstr .size (1..4096),
}

lez_node.start_node_response = {
    result: "started" / "already_running",
} / {
    error: lez_node.error,
}
```

Starts the node against a configuration and a storage directory.

```cddl
lez_node.stop_node_request = {}

lez_node.stop_node_response = {
    result: "stopped" / "not_running",
} / {
    error: lez_node.error,
}
```

Stops the node.

```cddl
lez_node.reset_storage_request = {
    config_path: tstr .size (1..4096),
}

lez_node.reset_storage_response = {
    result: "reset",
} / {
    error: lez_node.error,
}
```

Stops the node and deletes the store of the zone the configuration names,
without restarting it. Exposed by the node module only.

104. <a id="f-104"></a>Every function this RFP defines MUST belong to exactly
     one of the three categories of [Function categories](#function-categories),
     and the contract of [#96](#f-96) MUST state which. The user category holds
     every function defined under the headings above but `query_peers`. The
     operations category holds `query_peers` ([#93](#f-93)), and any function
     the module exposes that reports how the node runs in more detail than its
     status does, such as `query_metrics` ([#94](#f-94)). The administrative
     category holds `start_node`, `stop_node`, and `reset_storage`, together
     with any other function the module exposes that changes how the node runs,
     such as the FFI's `init_logger` were the module to expose it. A function
     added to the surface later MUST be assigned to a category when it is added.
     **[New]**
105. <a id="f-105"></a>Each function exposed through the node module MUST be
     individually grantable. Where the deployer's Logos Core access policy
     grants a caller a list of methods on the module, that caller MUST reach
     those methods and no others, and the module MUST be built so that such a
     grant is enforced against it rather than refused as unenforceable.
     **[New]**
106. <a id="f-106"></a>An operations or administrative function MUST refuse a
     call unless the caller is the host runtime, an operator of that runtime, or
     a caller the node's own configuration names for that category. The module
     MUST decide this itself, from the caller identity Logos Core gives it: it
     cannot see which grant admitted a call, since grants are enforced before
     the module runs and the caller document carries only a kind, a name, and an
     instance (`logos-protocol` `cpp/logos_module_impl.h:223-283`). The refusal
     MUST hold when no access policy is in force, when the policy leaves the
     module open to every caller, and when the runtime cannot identify the
     caller. Logos Core emits host, module, and unknown callers today, and
     specifies an operator kind it does not yet emit, so until it does the
     documentation MUST state which callers can start, stop, and reset the node,
     and a node MUST be configurable to start on load. **[New]**
107. <a id="f-107"></a>A user function MUST require nothing of a caller beyond
     admission to the module. A user function MUST NOT require a caller to
     register, to name a wallet, or to have made an earlier call, and the node
     MUST NOT hold state for a caller other than the subscriptions that caller
     opened. A cursor carries the position it resumes from, so a walk part way
     through holds nothing on the node. **[New]**
108. <a id="f-108"></a>A refused call MUST be reported as an outcome distinct
     from the not-found, invalid-argument, and backend failure outcomes of
     [#101](#f-101), the `refused` error, and MUST have no effect on the node. A
     caller does not retry it. **[New]**
109. <a id="f-109"></a>Calls from different callers MUST be served concurrently,
     up to a documented bound on calls in flight, and a slow call from one
     caller MUST NOT delay another's before that bound is reached. The module
     MUST run with Logos Core's concurrent dispatch, where a module dispatches
     one call at a time by default (`logos-module-builder`
     `docs/configuration.md`, `concurrency`), and the module wrapping the FFI
     sets nothing today. A cursor issued to one caller MUST NOT be advanced or
     invalidated by another caller's calls. **[New]**
110. <a id="f-110"></a>A subscription MUST belong to the caller that opened it,
     and MUST be named by an identifier no other caller can guess, so callers
     the runtime cannot tell apart, which arrive as one unknown caller, still
     cannot cancel or read one another's subscriptions. `unsubscribe` MUST
     cancel only a subscription whose identifier the caller holds, a caller's
     subscriptions MUST end when it disconnects, and one caller's cancellation,
     disconnection, or failure to keep up MUST NOT stop or delay delivery to any
     other caller. **[New]**
111. <a id="f-111"></a>Once the node is stopped, every call reaching the module
     from every caller MUST report the node as stopped, the `node_stopped`
     error, distinctly from a backend failure, and each open subscription MUST
     end with a final response saying that delivery has ended. **[New]**

#### Usability

1. <a id="use-1"></a>The delivery MUST include the LEZ node module and its CDDL
   contract ([#96](#f-96)), covering every function in Functionality, usable to
   build Logos modules for reading LEZ state, following the chain, and tracking
   deposits.
2. <a id="use-2"></a>The delivery MUST include a CLI that covers core
   functionality: read an account at current and at a pinned block, read a batch
   of accounts, read a block, read a transaction, read a transaction's effects,
   receipt, and status, read an account's balances, list an account's
   transactions with ordering and pagination, follow the blocks at `pending` and
   at `finalized`, follow a program's events, read the node's status, mode, and
   peer count, read the chain parameters and the fee market, list the mempool,
   list publications, and walk the encrypted notes of a block range. Every read
   MUST take the Bedrock status of [#97](#f-97). The CLI MAY have fewer features
   than the module but MUST support all essential operations.
3. <a id="use-3"></a>The delivery MUST include a worked deposit-tracking
   example, in the CLI or as a documented reference consumer, that follows the
   chain from a chosen start block, detects credits to a supplied set of public
   accounts, and reports each credit with its transaction hash, amount, block,
   and transaction status.
4. <a id="use-4"></a>The documentation MUST set out which state is readable and
   which is not for privacy-preserving transactions, including that a send
   published clear, from a public or a private sender, reveals its amount, its
   receiving private address, and its sender as presented, but not who controls
   that address, and that what a private account holds is not trackable from the
   node's data without its viewing key.
5. <a id="use-5"></a>The API MUST return clear, actionable error messages for
   every failure mode, each carrying the machine-readable value required by
   Functionality [#102](#f-102) alongside the text.
6. <a id="use-6"></a>The documentation MUST describe the semantics of every
   pagination parameter, including the exclusivity of the block bound, what a
   cursor guarantees across ingestion, and the behaviour when new data lands
   during a walk.
7. <a id="use-7"></a>The delivery MUST include a reference Logos Core access
   policy for each kind of deployment in
   [Who uses it, and how it is deployed](#who-uses-it-and-how-it-is-deployed): a
   user's own node, on the same machine as the wallet or another, a node in an
   integrator's infrastructure serving its users' wallets, and a node an
   operator offers as a service. Each MUST grant the user category to wallets,
   the operations category to the operator's monitoring, and the administrative
   category to no caller beyond those Functionality [#106](#f-106) admits.
8. <a id="use-8"></a>The documentation MUST describe how a deployer grants a
   caller a single function or a subset of the user category, such as the reads
   without `submit_transaction`.
9. <a id="use-9"></a>The documentation MUST describe the two Bedrock statuses
   for a caller: which blocks a `pending` read and a `finalized` read each see,
   what each guarantees, how long a block takes to become `finalized` on
   testnet, how a `pending` read reports that a block it saw was replaced, and
   which status a deposit tracker, a wallet building a transaction, and an
   explorer should each read at.

#### Reliability

1. <a id="rel-1"></a>A pinned account read at a given block identifier MUST
   return the same value on every call for as long as that block is the one at
   the identifier, which for a `finalized` block is for good, and MUST return
   the hash of the block it answered against so a caller can tell that a
   repeated read resolved to the same block.
2. <a id="rel-2"></a>A batch account read pinned to a block identifier MUST
   return a set of accounts consistent with one another as at that block, not a
   mixture of values read at different points during concurrent ingestion.
3. <a id="rel-3"></a>Pagination MUST remain valid across ingestion: a caller
   walking an account's transactions by cursor, or blocks, publications, and
   encrypted notes by position, neither skips nor repeats an entry that existed
   when the walk began.
4. <a id="rel-4"></a>A subscription consumer that disconnects and reconnects
   with its last processed block identifier and hash MUST receive every block
   after that position, with no gap and no assumption that the consumer was
   connected.
5. <a id="rel-5"></a>A stalled or lagging node MUST be detectable from any read,
   through the tips every response carries (Functionality [#98](#f-98)).
6. <a id="rel-6"></a>A stalled node MUST be reported as stalled by the status
   surface rather than serving stale reads silently.
7. <a id="rel-7"></a>The growth of history MUST be bounded by something the
   deployment controls rather than by the length of the chain. The node runs
   both as infrastructure an operator provisions and inside Basecamp on an end
   user's machine, where no one is watching a disk and no one can add one, so
   growth that a server operator would size for is a defect on a desktop. A
   deployment MUST be able to put a ceiling on the history the node retains,
   blocks, transaction bodies, effects, per-account indexes, and events, and
   reaching that ceiling MUST degrade what the retention floor reports rather
   than failing reads or exhausting the disk. The state the node validates
   against cannot be bounded this way, so the node MUST report its size and the
   documentation MUST project its growth.
8. <a id="rel-8"></a>A caller MUST NOT be able to degrade the node for the
   others through the user set. Every user function MUST bound what one call
   consumes, including the size of a batch, the limit of a page, the span of a
   range, the size of a response, and the number of subscriptions one caller
   holds open. A call past a bound MUST return the `invalid_argument` error, and
   the bounds MUST be documented. A whole-account read (Functionality
   [#15](#f-15), [#24](#f-24)) of an account whose actor states exceed the
   response bound MUST report the account as too large to read whole, the
   `too_large` error, and the caller reads it one program at a time
   (Functionality [#17](#f-17)).

#### Performance

1. <a id="perf-1"></a>The documentation MUST state the latency of every exported
   function, measured rather than estimated, against a stated chain size and
   transaction density.
2. <a id="perf-2"></a>Where a function's cost varies with something the caller
   controls or observes, the documentation MUST state what it varies with and
   report the worst case as well as the typical one. A read of a past block, a
   batch sized by the caller, and a page bounded by a limit each cost more than
   a read at the tip, and a caller choosing between them needs to know by how
   much.
3. <a id="perf-3"></a>A batch account read of N accounts MUST cost materially
   less than N single reads, and the improvement MUST be measured and documented
   for representative N.
4. <a id="perf-4"></a>The documentation MUST state the storage cost of
   persisting per-transaction effects: bytes per transaction, and projected
   growth against a stated block rate and transaction density.
5. <a id="perf-5"></a>Benchmarks MUST be reproducible from the test suite and
   run against a LEZ devnet or testnet node.
6. <a id="perf-6"></a>The documentation MUST state the latency of the user
   category under concurrent callers, measured against a stated number of
   callers and open subscriptions, alongside the single caller figures.
7. <a id="perf-7"></a>The documentation MUST state the size of the viewing key
   walk against the full blocks it is drawn from, in bytes per block and per
   encrypted note, and the time the node takes to serve a stated number of
   blocks of it, so a consumer restoring from a viewing key knows what fetching
   the walk from genesis costs.
8. <a id="perf-8"></a>The documentation MUST describe how one node scales with
   the deployments it serves, from a user's own node to one an integrator or an
   operator runs for many users: the callers, open subscriptions, and calls per
   second it sustains on stated hardware, and how its cost grows with the number
   of accounts its callers read and follow. Where a figure falls short of a node
   serving many users, the documentation MUST say so.

#### Supportability

This RFP states its requirements on the LEZ node module and its CDDL contract
([Assumption A9](#a9-the-api-defined-here-is-the-lez-node-modules-api)). How the
module is built is the implementation's: the node FFI it links today is a
private detail of the module, which this RFP neither requires nor forbids. "The
module" below means the LEZ node module together with its contract.

01. <a id="sup-1"></a>The module MUST be delivered for LEZ testnet 0.4: it MUST
    be built and tested against a LEZ devnet and against LEZ testnet 0.4.
02. <a id="sup-2"></a>End-to-end integration tests MUST run in CI against a
    local stack of a Logos Blockchain node and LEZ nodes, at least one in
    sequencer mode and one outside it, with the node module loaded in a Logos
    Core runtime and called through it. The repository's `docker-compose.yml`
    assembles a dev Bedrock node, a sequencer committee, and the node's RPC
    service today, without the FFI, the module, or a Logos Core runtime, and
    LEZ's own CI runs a Bedrock node with its test fixtures rather than the
    compose file. The tests of the module and its access control MUST drive a
    Logos Core runtime with several callers, as `logos-capability-module` drives
    its module through `logosctl` in doctests. Every function and outcome whose
    behaviour differs by mode MUST be tested in each mode, including a call that
    is not available in a node's mode (Functionality [#99](#f-99)).
03. <a id="sup-3"></a>CI MUST build and test the module, and MUST be green on
    the default branch of every repository the work lands in. The module that
    wraps the FFI today has no build or test workflow.
04. <a id="sup-4"></a>Every hard requirement in Functionality, Usability,
    Reliability, and Performance MUST have at least one corresponding test.
05. <a id="sup-5"></a>Tests MUST cover, at minimum: a read of an identifier the
    state holds no record for; a pinned read at a block before and after a state
    change; a batch read spanning recorded and unrecorded accounts; effects for
    a public transaction, for the public leg of a privacy-preserving
    transaction, and for a transaction whose action reverted; membership proofs
    for a commitment and a message position the set holds and ones it does not;
    a clear publication from a public and from a private sender, and a sealed
    one, in the listing; the viewing key walk over a range, returning every
    encrypted note the range's transactions carry, for a regular private
    account, a private PDA, and a shared private account; a `pending` read and a
    `finalized` read while a block is inscribed but not final, and a `pending`
    read across that block being replaced; each documented transaction status; a
    submission to a node in sequencer mode and to one outside it, and to one
    with no peers; a subscription resumed from a stored position across a
    disconnection, and one resumed from a position whose hash does not match; an
    administrative call and an operations call from a caller holding no grant,
    with no access policy in force and with a policy that leaves the module
    open; and two callers holding subscriptions when one of them cancels.
06. <a id="sup-6"></a>A README MUST document end-to-end usage: building the
    module, the configuration and storage directory `start_node` requires, and
    step-by-step instructions for every operation via the CLI.
07. <a id="sup-7"></a>The delivery MUST include documentation in repo for the
    module and its contract, covering the developer integration journey for
    reading state, tracking deposits, and confirming transactions.
08. <a id="sup-8"></a>The delivery MUST include documentation in repo for the
    CLI, covering the core operator journey.
09. <a id="sup-9"></a>The delivery MUST include a get started guide that takes a
    developer from an empty repository to a working Logos Core module calling
    the LEZ node module, covering the client generated from the module's
    contract, reading LEZ state through it, following the chain, and running the
    result against a devnet, with the node in the same runtime and reached
    through peering. A module calls another through its contract rather than
    linking its library, so the guide MUST NOT link a library of the node. The
    README required above documents building and operating what this RFP
    specifies; this documents building something else on top of it.
10. <a id="sup-10"></a>The delivery MUST include a full API reference for the
    node API, published per version as the Logos Storage module publishes its
    [API reference](https://logos-co.github.io/logos-storage-module/latest/api_reference.html),
    covering every module method and event, its parameters, its return type, its
    error values, its set, and the node modes it is available in. The reference
    MUST be generated from the module contract required by Functionality
    [#96](#f-96) rather than maintained by hand, so it cannot drift from the
    surface it describes.
11. <a id="sup-11"></a>The delivery MUST include a design decision log in repo
    recording each major design decision and choice: the context, the options
    considered, the option chosen, and the rationale and trade-offs. The log
    MUST be updated as decisions are made and MUST be current at each milestone
    delivery.
12. <a id="sup-12"></a>The delivery MUST include a plan, agreed with the Logos
    Core maintainers and recorded in the design decision log, for how an
    application uses several zones while Logos Core runs one instance of each
    kind of module: whether through several instances of the node module, one
    per zone, or through one node serving several zones, and what an application
    sees, and for how long, when it moves between zones.

#### + Privacy

1. <a id="priv-1"></a>A function added by this RFP MUST NOT return private state
   in the clear, or a viewing key, or any value from which either can be
   recovered without the viewing key, and a function MUST NOT accept a viewing
   key, a nullifier key, or a decrypted note. Private post-states MUST remain
   readable only by the viewing key holder, who decrypts them on its own side.
2. <a id="priv-2"></a>The effects and status surfaces MUST NOT report, for a
   privacy-preserving transaction, any linkage between its private inputs and
   its private outputs beyond the commitments and nullifiers already published
   on-chain.
3. <a id="priv-3"></a>The delivery MUST include a privacy properties document
   covering: what a third party reading the node can learn about a public
   transaction, about the public leg of a privacy-preserving transaction,
   including the address its fee was paid from under either way of paying it,
   and about its private leg; what a send published clear reveals, from a public
   and from a private sender; which of those boundaries are enforced by
   encryption and which by the API declining to expose data it holds; and what
   an operator running their own node can see that a remote caller cannot. The
   document MUST state that the mempool listing shows, before inclusion, the
   private addresses a transaction binds, which its body already carries in
   clear. What a viewing key holder can learn belongs in the LEZ wallet API's
   documentation.
4. <a id="priv-4"></a>A function MUST NOT disclose one caller's activity to
   another. Which accounts, commitments, and transactions a caller asked about,
   which subscriptions it holds and with what filter, and which caller submitted
   a transaction MUST NOT be readable by any other caller through this surface.
5. <a id="priv-5"></a>The node MUST NOT keep a record tying a caller identity to
   what that caller read or submitted beyond what serving the call requires. Any
   logging that does so MUST be off by default, and the operator turns it on as
   Functionality [#95](#f-95) defines. This binds the node; what Logos Core
   records of each access decision is covered under Privacy [#6](#priv-6).
6. <a id="priv-6"></a>The privacy properties document MUST state what the
   operator of a node shared by several wallets learns about each of them from
   its calls: the accounts and commitments it reads, the transactions it
   submits, and the caller identity those arrive under. It MUST also state what
   Logos Core records about calls to the node, its draft specification giving
   every access decision an audit record.
7. <a id="priv-7"></a>A viewing key consumer MUST be able to follow its accounts
   through calls that name none of them. The viewing key walk, the publication
   listing, and the subscription delivering the walk MUST take positions only,
   and the documentation MUST identify every user function that takes an account
   identifier, a commitment, or a private address, so a consumer knows which
   calls disclose what it holds.

### Soft Requirements

If possible.

#### Functionality

1. <a id="soft-1"></a>The API SHOULD expose a subscription filtered to a
   supplied set of accounts, delivering only blocks containing a transaction
   that touches one of them. Most surveyed chains that offer a block
   subscription also offer a filtered one; Bitcoin and Zcash push unfiltered
   streams only
   ([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).
2. <a id="soft-2"></a>The API SHOULD expose a batch transaction read, taking a
   list of transaction hashes and returning a result per hash in request order.
3. <a id="soft-3"></a>The API SHOULD expose a block-scoped effects query
   returning the diffs for every transaction in one block, so a consumer
   following the chain reads one result per block rather than one per
   transaction.
4. <a id="soft-4"></a>The node SHOULD support retention policies beyond a single
   ceiling, such as keeping transaction history for accounts a deployment cares
   about while discarding the rest, so a desktop deployment holds what its user
   needs rather than the most recent window of everything.
5. <a id="soft-5"></a>The API SHOULD report the tips recent transactions chose,
   per block, so a caller chooses a tip against what was paid, should inclusion
   come to depend on the tip. Receipts record the tip each transaction paid
   (Functionality [#77](#f-77)), but for a proof-committed fee that figure
   includes a surplus kept rather than refunded, so this reads the tip each
   transaction body declares.
6. <a id="soft-6"></a>The API SHOULD expose a read of the commitment set that
   names no commitment: its frontier at a given block, with each commitment the
   viewing key walk (Functionality [#31](#f-31)) lists carrying its position, so
   a wallet keeps its own membership witnesses up to date and spends without
   naming to the node the commitments it holds, as a Zcash light client does
   from the tree state. Functionality [#36](#f-36) names them (Functionality
   [#38](#f-38)).

### Out of Scope

The following are explicitly excluded from this RFP:

- **Wallet and key management.** Key handling, derivation, watch-only address
  derivation, viewing keys, and signing belong to the LEZ wallet API, a separate
  component outside this RFP. That includes how a viewing key is exported and
  imported, a view-only mode for the wallet, which LEZ does not have today, and
  whether a group of controllers should hold a viewing secret of its own rather
  than a per-account viewing key derived from its master secret. Scanning the
  chain with a viewing key, recovering a particular account's states, and
  documenting what a viewing key holder can establish are wallet work too, since
  each is a process run for an account the caller holds keys for. This RFP
  requires only that the node serve, for every account alike, what a viewing key
  consumer decrypts.
- **Transaction construction and signing.** Building a transaction and signing
  it run through the wallet path (`core_ffi` and `lez_core`). This RFP requires
  the node to accept a signed transaction and admit or relay it, and nothing
  more: it does not construct, sign, or execute one. Verifying a signature, and
  encoding or decoding transaction bytes, which several surveyed chains offer on
  the node, belong with construction in the LEZ wallet API.
- **The protocol between LEZ nodes.** Relaying a submitted transaction to the
  nodes that sequence, gossiping transactions, any propagation of blocks, and
  coordinating the sequencers of a zone run over the LEZ peer-to-peer network,
  and their messages are not part of this API. This API is what a wallet or
  application calls on one node, and it reports the node's connectivity
  (Functionality [#92](#f-92), [#93](#f-93)) rather than exposing the protocol.
  That the node takes part in the network is part of the work (see
  [Assumption A3](#a3-following-the-chain-serving-its-history-and-sequencing-are-one-node)).
- **Operating sequencer mode.** The reads and actions specific to running a
  sequencer, such as its committee, approvals, slashing, and cross-zone dead
  letters, are not part of this API. A node in sequencer mode serves the API
  defined here, and how it is operated as a sequencer is outside it.
- **Decoding the actor states of other programs.** The node decodes the native
  balance and the balances of builtin token programs (Functionality
  [#20](#f-20), [#21](#f-21)). Any other program's actor state is returned as
  bytes, and decoding it from the program's IDL belongs to the SDK, as Solana
  leaves the accounts of programs its node does not know to the client.
- **Simulation, and the cost estimate that comes with it.** Executing an
  unsubmitted transaction to see what it would do belongs to the wallet API, for
  both public and privacy-preserving transactions, for the reason given in the
  Design Rationale. Nothing in this RFP executes a transaction a caller
  supplies. The node serves the fee market a caller prices against
  (Functionality [#88](#f-88) and [#89](#f-89)), not a fee for a given
  transaction.
- **The other node functions the surveyed chains offer that LEZ lacks**
  ([Appendix: Blockchain API and SDK Ecosystem, section 4.1](../appendix/blockchain-api-sdk-ecosystem.md#41-absent-on-both-targets)).
  A read-only call executes program code, and is simulation by another name: a
  caller reads the state through the account reads and executes on its own side.
  A pre-submission acceptance check is the wallet's own simulation together with
  the typed rejection of Functionality [#46](#f-46). Waiting for a transaction
  to reach a chosen status is composed from the status query ([#39](#f-39)) and
  the block subscription ([#57](#f-57)) rather than offered as a call that
  blocks. Batch submission is not required: each transaction is submitted on its
  own.
- **Hosting Logos Core in an application, and reaching a node on another
  machine.** The Logos Core host libraries are a separate component outside this
  RFP, and a node elsewhere is reached through Logos Core peering, which carries
  the module's contract ([#96](#f-96)). This RFP defines the functions, not how
  a call to them is carried.
- **The Logos Blockchain node and wallet APIs.** Each is a separate component
  outside this RFP.
- **Reaching inside a node.** A consumer reaches a LEZ node only through the API
  defined here. Whatever a node is made of internally, those interfaces are not
  an API this RFP or any other in this set defines. Every read an integrator
  needs is required of the node above, whichever part of it holds the data
  today.
- **Pairing, enrolment, and authentication of callers.** Pairing a wallet's
  runtime with a node on another machine, enrolling the runtimes of many users,
  and token based access are delivered by the Logos Core framework. This RFP
  relies on the caller identity Logos Core gives each call, and sorts the node's
  functions into the categories a deployer grants
  ([Function categories](#function-categories)).
- **Identifying wallets or users.** The node identifies a caller as Logos Core
  does and no finer. It issues no API keys, keeps no user accounts, and applies
  no quota or billing per user; the per-call and per-caller resource bounds of
  Reliability [#8](#rel-8) protect the node rather than meter its users. Which
  callers a deployment admits, and which functions it grants them, is decided in
  the deployer's Logos Core access policy.
- **Extending the program event system.** Events exist from program execution
  through capture by the node to the node RPC's `getEvents` and
  `subscribeToEvents`. The FFI exports `query_events` and the module does not,
  and the subscription stops at the RPC. This RFP requires both to reach
  applications (Functionality [#12](#f-12) to [#14](#f-14), [#63](#f-63),
  [#64](#f-64), [#100](#f-100)) and does not change what an event is or which
  events are captured. The other requirements neither depend on events nor
  duplicate them, for two reasons: events do not cover execution inside a proof,
  and their availability depends on how an operator configured the event filter.
- **Tracking deposits into private accounts without a viewing key.** Private
  post-states are encrypted and readable only by the viewing key holder, so this
  is foreclosed by the design of the chain rather than by the scope of this RFP.
  A third party sees the amount and receiving address of a send published clear
  and nothing after it; a viewing key holder reads the rest, scanning the
  viewing key walk above on its own side.
- **Deploying and operating a node.** Mainnet deployment is not part of this
  RFP: whether, when, where and by whom a node exposing this API is deployed or
  operated, which callers a deployment admits, and who holds the administrative
  category are matters for whoever deploys it.

## ⚠ Platform Dependencies

Most capabilities required above are already present on the node RPC and
unexported, already computed by the node and discarded, or derivable from data
the node's store already holds. The exceptions are what the target architecture
needs that the node does not provide today. Closing them on the LEZ side is part
of the work (see
[Assumption A3](#a3-following-the-chain-serving-its-history-and-sequencing-are-one-node)),
and each is listed so the work can be planned and costed.

- **Peer-to-peer participation in every mode.** A node relays a submission
  (Functionality [#45](#f-45)), observes the mempool ([#51](#f-51)), and reports
  its peers ([#92](#f-92)) over the LEZ peer-to-peer network. Today only the
  sequencer joins that network: transaction gossip runs on a topic per zone in
  the sequencer's gossip actor
  (`lez/sequencer/actors/gossip/src/actor.rs:338-340`), and the node joins no
  peer-to-peer network at all. The sequencer service can already run without
  sequencing, as the read-only sequencer the wallet embeds
  (`lez/core_ffi/src/sequencer/service.rs`), following the chain at `pending`
  and joining gossip, which is the closest thing today to a node outside
  sequencer mode.
- **One node in both modes.** The node and the sequencer run today as separate
  services, each with its own RPC, the node holding the history, `finalized`
  blocks only, and the sequencer the blocks a `pending` read sees. A node in
  either mode serves the API defined here from both. A sequencer's mempool is
  also only pushed to today (`lez/mempool/src/lib.rs:66-92`), so listing it is
  work in the sequencer. The design decision log (Supportability [#11](#sup-11))
  records how the two are brought together.
- **`pending` reads.** The node reads blocks only once they are final on L1
  (`lez/chain_state/src/zone_indexer.rs:54-86`) and has no path for undoing a
  block. Reads at `pending` ([#97](#f-97)), the subscription to `pending` blocks
  ([#59](#f-59)), and the transaction statuses short of `finalized`
  ([#39](#f-39)) need the node to follow inscriptions at the L1 tip and to
  handle a replaced block, as the sequencer service does
  (`lez/chain_state/src/chain.rs:1-3`).

Access control and the contract rest on Logos Core in several places.

- **The contract format.** The API is defined in the CDDL profile of the draft
  Logos Core specification
  ([Assumption A9](#a9-the-api-defined-here-is-the-lez-node-modules-api)), which
  Logos Core does not yet implement: modules publish LIDL contracts today, with
  JSON values. The CDDL contract, its description at run time, clients generated
  from it, its invocation-scoped events, and the schema root peering compares
  are in that draft.
- **One instance per module.** Logos Core runs one instance of each kind of
  module, while a node follows one zone
  ([Assumption A10](#a10-each-lez-node-follows-one-zone)), so an application
  using several zones restarts the node module to move between them today.
  Supportability [#12](#sup-12) asks for the plan.
- **Call identifiers.** Logos Core gives each transport request an identifier
  the callee echoes (`logos-lips` draft at `6cac8335`,
  `spec-module-transport.md:335-336`), records an authority decision identifier,
  and keeps call audit records naming the caller
  (`spec-module-capability-authority.md:538-539`, `:1067-1084`). The node issues
  no correlation identifier of its own and logs against those ([#95](#f-95)), so
  it needs Logos Core to give a module the identifier of the call it is serving,
  and the caller a value it can quote to an operator, which the draft
  specifications do not yet state.
- **A dependency needed in one mode.** A node in sequencer mode needs a Logos
  Blockchain wallet module to inscribe with, and a node outside it does not
  ([Assumption A2](#a2-a-sequencer-is-a-lez-node-in-sequencer-mode)). The module
  builder's `optional_dependencies` loads a module when it is installed and
  never requires it; whether a node can rely on it to bring the wallet module up
  for sequencer mode alone is to be settled with Logos Core.
- **Caller identities.** The caller document a module receives names a host, a
  module, a derived caller, an operator, or an unknown caller (`logos-protocol`
  `cpp/logos_module_impl.h:223-283`), and only host, module, and unknown are
  emitted today. A call authorised by an operator token over a plain connection
  arrives as unknown. Functionality [#106](#f-106) admits the host, an operator,
  and the callers the node's configuration names, so who can operate the node
  from elsewhere waits on operator identities.
- **Per method grants.** The access policy admits a caller to a module today.
  Narrowing a caller to a list of methods is in development
  ([logos-liblogos#232](https://github.com/logos-co/logos-liblogos/pull/232),
  [logos-capability-module#37](https://github.com/logos-co/logos-capability-module/pull/37))
  and is what Functionality [#105](#f-105) depends on. Functionality
  [#106](#f-106) does not depend on it, deciding on the caller identity and the
  node's own configuration instead.
- **Callers on another runtime.** A wallet on another runtime reaches the node
  through Logos Core peering, which is also in development
  ([logos-liblogos#230](https://github.com/logos-co/logos-liblogos/pull/230)),
  and which identifies each peered runtime by its own keys. Serving many users'
  runtimes from one node through peering is still in design. The documentation
  states how a per method grant is enforced for such a caller where the platform
  does not enforce it.
- **Events and lifecycle per caller.** Functionality [#110](#f-110) and Privacy
  [#4](#priv-4) require delivery per caller, and [#110](#f-110) requires a
  disconnected caller's subscriptions to end. A Logos Core module event
  published in the ordinary way reaches every subscriber of the module. The
  draft's invocation-scoped events, on which the subscriptions above are
  modelled, reach only the invocation that watches for them, and stop with its
  response, its cancellation, or the loss of its route
  (`spec-module-runtime.md:1173-1184`, `:1200-1201`). Until Logos Core
  implements them, the documentation states how a subscription is delivered to
  the caller that opened it and to no other, and how the module learns that a
  caller has gone.

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
  inventory of the LEZ node FFI, RPC, and module, the sequencer RPC and FFI,
  `core_ffi` and the `lez_core` module, and the L1 bindings, routes, and module

- [Appendix: Blockchain API and SDK Ecosystem](../appendix/blockchain-api-sdk-ecosystem.md):
  34 API functions across nine established chains, with transports, SDK
  languages, response shapes, and per-function gap notes for LEZ

- [logos-execution-zone](https://github.com/logos-blockchain/logos-execution-zone):
  the LEZ node, sequencer, and wallet

- [`logos-execution-zone-module`](https://github.com/logos-blockchain/logos-execution-zone-module):
  `lez_core` Logos Core module

- [Logos glossary](https://docs.logos.co/get-started/glossary): zone, channel,
  Bedrock, Basecamp, module, viewing key, and note

- [LEZ documentation](https://docs.logos.co/lez): the sequencer and node roles,
  and the separation of public and private state

- [Logos Documentation](https://github.com/logos-co/logos-docs)

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## 💬 Feedback

This RFP is not open for proposals. Integrators with questions about the API it
describes, or with needs it does not meet, are invited to raise them in the
repository's **Discussions**.
