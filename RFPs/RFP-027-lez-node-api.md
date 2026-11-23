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
Each is written against the node FFI function that backs it. It is the
equivalent of the `eth` namespace on Ethereum's JSON-RPC: it answers what
happened and what the current state is, and it accepts a signed transaction for
the network, holding no keys and signing nothing.

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

In that mode it also holds the pending set it builds blocks from, produces
blocks, and inscribes them on the Logos Blockchain.

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
sequencer mode reports it as not available in its mode ([#98](#f-98)).

**Basis:** the integration guide, which has a LEZ node acting as a sequencer
also use a Logos Blockchain wallet. **Rests on it:** the node's mode in status
([#35](#f-35)), admission and relay ([#37](#f-37) to [#39](#f-39)), the pending
set ([#42](#f-42)), the produced level ([#29](#f-29)), and availability by mode
([#98](#f-98)).

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
([#38](#f-38) and [#39](#f-39)), and every read of account state, membership
proofs, and publications ([#14](#f-14) to [#28](#f-28), [#85](#f-85) to
[#88](#f-88)).

#### A5. LEZ nodes talk to each other over a p2p protocol outside this API

LEZ nodes communicate with each other over the LEZ p2p protocol, which is not
part of this API.

Relaying a submitted transaction to the nodes that sequence and gossiping it run
over that network (see [Out of Scope](#out-of-scope)). What a caller can learn
about the node's place in it, such as how many peers it has, is part of this API
([#83](#f-83) and [#84](#f-84)).

**Basis:** the integration guide, and the sequencer's gossip today. **Rests on
it:** relay ([#37](#f-37) and [#39](#f-39)), the observed pending set
([#43](#f-43)), and the peer reads ([#83](#f-83) and [#84](#f-84)).

#### A6. A node outside sequencer mode sees pending transactions on gossip

A node outside sequencer mode observes pending transactions on the LEZ
transaction gossip, and serves those it has not yet seen inscribed as its
pending set.

Transactions sequencers gossip travel on a peer-to-peer topic per zone, which
any node following the zone can subscribe to, so a wallet on the user's own node
can still abandon a proof that a pending transaction invalidates.

**Basis:** the transaction topic of the gossip actor today
(`lez/sequencer/actors/gossip/src/actor.rs:338-340`). **Rests on it:** the
pending set outside sequencer mode ([#42](#f-42) and [#43](#f-43)) and the
received level ([#29](#f-29)).

#### A7. LEZ blocks are not propagated over the LEZ p2p protocol

LEZ blocks are not propagated between LEZ nodes over the LEZ p2p protocol. Every
node, sequencers included, learns of a block only from its inscription on the
Logos Blockchain.

Sequencers learn of each other's blocks through the L1 alone today, gossip
carrying transactions and approvals
(`lez/sequencer/actors/gossip/src/lib.rs:4-6`), and no plan to change that was
found. A sequencer inscribes a LEZ block in an L1 block, and the block is final
once that L1 block is irreversible, about an hour after inscription on testnet
today.

**Basis:** the code at the commit read, and the absence of any plan found.
**Rests on it:** the status levels ([#29](#f-29)), the head subscription
([#50](#f-50)), and the head of
[Assumption A8](#a8-every-node-holds-a-head-as-well-as-a-final-chain-and-reads-default-to-the-head).
**If invalidated:** a node learns of a block before its inscription, and
[#29](#f-29) gains a level for that.

#### A8. Every node holds a head as well as a final chain, and reads default to the head

Every LEZ node holds a head, the blocks it has seen inscribed whether final or
not, beside a final chain, and every read answers against the head unless the
caller asks for the final chain.

An L1 reorganisation or a competing inscription can still replace the head;
nothing replaces the final chain. A transaction's status moves through being
received, inscribed, and finalised. The sequencer service already keeps both
tiers and answers against the head (`lez/chain_state/src/chain.rs:1-3`), while
the node today reads only to finality
(`lez/chain_state/src/zone_indexer.rs:54-86`).

**Basis:** the sequencer service's head and final tiers, and a finality delay
too long for a wallet to build against the final chain alone. **Rests on it:**
the read level ([#96](#f-96)), the tips every read reports ([#97](#f-97)),
status ([#29](#f-29)), the head subscription ([#50](#f-50)), the chain tip
([#71](#f-71)), the fee state ([#79](#f-79) and [#80](#f-80)), and the recovery
binding lookup ([#87](#f-87)).

#### A9. The API defined here is the LEZ node module's API

The API this RFP defines is the API the LEZ node module is expected to expose,
and, as the new Logos Core architecture has every module do, it comes with a
CDDL schema describing it.

Callers use clients generated from that schema, whether they run in the same
runtime as the node or reach it from another through Logos Core peering. Modules
publish LIDL contracts today, and the module publishes this API in whichever
format Logos Core supports when it ships ([#94](#f-94)). The requirements below
are written against the node FFI functions that back each module method, and a
capability reaches a caller only once the module carries it ([#99](#f-99)). See
[Appendix: Logos API Surfaces](../appendix/logos-api-surfaces.md) for the
as-built inventory of every layer.

**Basis:** the integration guide and the draft Logos Core specification. **Rests
on it:** the module contract ([#94](#f-94)), module parity ([#99](#f-99)),
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
network identity ([#72](#f-72)), storage reset, which removes the store of the
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
transaction to reach a level it is willing to act on. A wallet also needs what
an exchange never asks for: the proofs it spends with, the pending set it checks
while proving, and the publications and encrypted notes it receives through.

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
[Functionality](#functionality) [#79](#f-79) and [#80](#f-80) require the node
to serve the fee market, and [#96](#f-96) lets every read answer at the head, so
a wallet simulating on its own side reaches every input it needs, at the state
it is building on, through the node API.

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
Ethereum library on npm (42,475,870 downloads from 23 October to 21 November
2026, against 20,757,782 for ethers), keeps a public client, which reads from a
node over HTTP, WebSocket, or an injected provider, apart from a wallet client
and its local accounts, which hold the keys and sign. A transaction is signed in
the library and handed to the node raw.

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
is holding. [#6](#f-6) scopes the balance deltas the effects query returns to
public transactions and the public leg of privacy-preserving ones, which is what
an integrator crediting deposits reads: a deshield into a public account is
reported, and a transfer between private accounts is not. Because the public leg
of a privacy-preserving transaction is readable only by executing it, [#5](#f-5)
persisting what ingest computed is what makes that leg reportable at all.

The wallet is the other consumer, and there the work is on this surface rather
than on the capability. [#26](#f-26) and [#27](#f-27) require the membership
proofs and the root they are proven against, which a wallet needs to spend. The
publication listing and recovery-binding reads under Private message delivery
are what a wallet needs to receive. The sequencer service answers all of these
today, and its own RPC marks them for removal once the wallet reads from the
node (`lez/sequencer/service/rpc/src/lib.rs:64-65`), while the state the node
already holds carries the data (`lee/state_machine/src/state/mod.rs:357-438`).
What is new is serving them from the node. [#62](#f-62) requires the
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
call that arrives through Logos Core. Whoever links the node FFI library and
holds the handle is the node's host and can call every exported function, so the
requirements below bind the node module, which is the surface a wallet reaches.

**Callers are no finer than Logos Core makes them.** Two wallets in two modules,
two UI plugins, or two runtimes are two callers. Two wallets behind one calling
module are one caller, and the node cannot tell them apart. Because the node
keeps nothing per wallet, it does not need to. [Out of Scope](#out-of-scope)
records that the node issues no credentials of its own.

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
  as the peer list with other operators' addresses ([#84](#f-84)) and any
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

[Functionality](#functionality) [#81](#f-81) requires what the node retains to
be exposed, and [#82](#f-82) requires a read below it to be answered as such,
distinctly from a read for data that never existed.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

Each requirement carries a readiness marker describing its starting point in the
current LEZ codebase. **Ready** means the capability is already exported by the
node FFI and only a specification, test, or documentation obligation remains.
**Ready, not exposed** means it exists in the node service, RPC, or store, or
the sequencer service serves or computes it from state the node also holds, but
the node FFI does not export it, so the work is exposure. **Computed, not
persisted** means the node derives the data during ingestion but neither stores
nor exposes it, so the work is persistence plus exposure. **New** means the
capability does not exist today, whether or not the data it would serve does. A
requirement whose behaviour depends on the node's mode says so. The markers
indicate effort, not priority: every hard requirement is required regardless of
its marker. When not already implemented, the function signatures below should
be seen as suggestions; the description of functionality underneath is the
requirement.

The markers describe the FFI on the LEZ `dev` branch, so a marker of **Ready**
means ready in the FFI. The Logos Core module that wraps the FFI today is built
against LEZ `v0.3.1`, from before the FFI's account read became optional and its
account record moved to actor states, so a capability ready in the FFI reaches
the node module only once the module is built against the testnet 0.4 FFI, which
[#99](#f-99) requires.

Every signature below elides a first argument, the node handle returned by
`start_node`. See Surface-wide obligations.

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

1. <a id="f-1"></a>The query returns the stored transaction for a known hash and
   an absent `FfiOption` for a hash no indexed block holds, keeping the two
   distinct from a backend failure. **[Ready]**
2. <a id="f-2"></a>The returned transaction discriminates public and
   privacy-preserving bodies, exposing for each the fields the wire type carries
   (`lez/indexer/service/protocol/src/lib.rs:254-441`). A program is deployed by
   a public transaction to the program loader rather than by a transaction kind
   of its own. **[Ready]**
3. <a id="f-3"></a>The returned transaction carries every field of the
   transaction as the chain holds it, so a caller can recompute its hash. The
   wire type drops a privacy-preserving transaction's program image claims
   (`lez/indexer/service/protocol/src/convert.rs:599-602`), so a transaction
   carrying any cannot be reproduced from what is returned. **[New]**
4. <a id="f-4"></a>The API defines what a hash naming more than one transaction
   returns. The fee transaction that closes each block carries no nonce or block
   identifier (`lez/common/src/transaction.rs:355-395`), so in a run of blocks
   with no revenue and the same producer it repeats byte for byte, and the index
   from hash to block keeps only the latest
   (`lez/storage/src/indexer/write_atomic.rs:40-46`). Every read by hash states
   which transaction it answers for, and a transaction is also addressable by
   its block identifier and its index in the block. **[New]**

**`query_transaction_effects(hash) -> PointerResult<FfiOption<FfiStateDiff>, OperationStatus>`**

Returns the state diff a transaction produced, read from storage rather than
recomputed on the caller's behalf.

5. <a id="f-5"></a>The FFI exposes a per-transaction effects query that, given a
   transaction hash, returns the state diff that transaction produced: the
   accounts whose public state changed, with their pre-state and post-state
   nonces and, for each actor state that changed and only those, its pre-state
   and post-state value; the new commitments; the new nullifiers; the
   publications produced, with their positions; the recovery bindings
   introduced; and the program the public root call invokes, absent when
   execution starts privately. A diff as computed today carries each touched
   account's whole post-state, untouched actor states included
   (`lee/state_machine/src/validated_state_diff/mod.rs:33-42`), and any party
   can write actor states of any size onto any account, so reporting whole
   states would let a third party inflate every effects response for an account
   it targets. The diff is the one the chain applied when the transaction
   settled, as the node computes it on ingest
   (`lez/chain_state/src/apply.rs:391-812`), not one recomputed for the caller.
   A transaction paying a declared fee settles as separate diffs for its fee
   reserve, its action, and its refund (`lez/chain_state/src/apply.rs:654-812`);
   one paying from a reserve its proof commits to carries the reserve as a
   crossing inside its action, with any refund separate
   (`lez/chain_state/src/apply.rs:485-592`). Each diff is reported with the
   state it applied to. **[Computed, not persisted]**
6. <a id="f-6"></a>The effects query returns a balance delta per affected
   account, expressed as pre-state and post-state values, for public
   transactions and for the public leg of privacy-preserving transactions. The
   native balance is the account's actor state at the native token program
   (`lez/indexer/service/protocol/src/lib.rs:106-127`), and a token held through
   another program is that program's actor state, so the delta is reported per
   actor state rather than as a single balance. **[Computed, not persisted]**
7. <a id="f-7"></a>The fee is reported separately from the balance changes the
   transaction itself caused, naming the account charged, the amount taken, and
   any amount refunded. An exchange reconciling a withdrawal has to tell what it
   sent from what it paid to send it, and a caller that has to reconstruct the
   fee by arithmetic on the remaining deltas is reimplementing the chain's fee
   rules. A public transaction declares a payer, a gas limit, a tip, and a cap
   on the fee; the payer need not be an account the transaction otherwise acts
   on, but it must sign (`lee/state_machine/src/fees.rs:34-52`). A
   privacy-preserving transaction pays either from a reserve its proof commits
   to, together with a tip it commits, or through a fee declaration signed by
   one of its public signers, as a public transaction does, and the effects
   query reports which. A surplus left in a proof-committed reserve has one of
   three fates: refunded to the target the proof names, published to the payer's
   bound alias as a private credit, or, when it is no more than
   `MIN_PRIVATE_REFUND` or there is nowhere to send it, kept as tip
   (`lez/chain_state/src/apply.rs:575-592`), and the query reports which and
   where. The base fee, tip, and refund each transaction paid are in its receipt
   **[Ready]**; the account charged, which way a privacy-preserving transaction
   paid, and the fate of its surplus are computed when it settles and then
   discarded **[Computed, not persisted]**. A fee-exempt system transaction is
   reported as exempt rather than as one that paid nothing.
8. <a id="f-8"></a>The gas a transaction consumed is reported alongside the fee,
   in each dimension the chain meters, execution and storage
   (`lez/programs/fee/core/src/lib.rs:24-29`), so a caller comparing what it
   budgeted against what it paid can tell a transaction that ran long from one
   that paid a high price per unit. Both kinds of transaction are metered: a
   privacy-preserving transaction is charged a fixed verification cost, storage
   on its serialized size with its proof counted at a fixed length, and the
   metered execution of its public part
   (`lez/chain_state/src/apply.rs:515-544`). The receipt carries both
   dimensions, and records a fee-exempt transaction as all zeros
   (`lez/common/src/block.rs:59-95`); the API reports exemption as such rather
   than as a transaction that consumed zero, which a charged transaction never
   does, since it always meters storage. **[Ready]**

**`query_transaction_receipt(hash) -> PointerResult<FfiTransactionReceiptOpt, OperationStatus>`**

Returns the receipt the block records for a transaction: whether its action
applied, and the gas, base fee, tip, and refund it was charged.

09. <a id="f-9"></a>The FFI returns a transaction's receipt, exporting what the
    node RPC's `getTransactionReceipt` serves
    (`lez/indexer/service/rpc/src/lib.rs:100-105`,
    `lez/indexer/ffi/src/api/query.rs:359-407`). The wallet already reads a
    finished transaction's outcome this way, from the sequencer service today
    (`lez/sequencer/service/rpc/src/lib.rs:92-97`). **[Ready]**
10. <a id="f-10"></a>A transaction the chain records without a receipt, the fee
    and clock transactions that close each block
    (`lez/common/src/block.rs:100-105`), is reported as having none, distinctly
    from a hash no indexed block holds. Both return an absent receipt today.
    **[New]**

**`query_events(from_block, to_block, tx_hash, program_account_id, selector) -> PointerResult<FfiVec<FfiEventRecord>, OperationStatus>`**

Returns the program events matching the filter. A non-null `tx_hash` makes the
call a point lookup and the block range is ignored; otherwise the range runs
from `from_block` to `to_block`, defaulting to the indexed tip.

11. <a id="f-11"></a>The query returns each matching event record with its block
    identifier, transaction index, transaction hash, emitting program, selector,
    and data payload **[Ready]**, and its index among the transaction's events,
    so a consumer deduplicates and resumes by event **[New]**.
12. <a id="f-12"></a>A request the query cannot answer is reported as the
    outcome [#100](#f-100) and [#102](#f-102) define for its case rather than as
    an empty result: a range spanning more than `MAX_EVENT_QUERY_BLOCK_SPAN`
    blocks, or inverted, as an invalid argument; a bound past the indexed tip as
    the lagging case; a range outside the node's event-filter history as data
    the node does not hold, as a read below the retention floor is; and a point
    lookup for a hash no indexed transaction carries as not found. Each returns
    `InvalidArgument` today (`lez/indexer/ffi/src/api/query.rs:553-633`). The
    query also bounds the size of its response, as the node RPC does at 8 MiB
    (`lez/indexer/service/src/service.rs:28`), and reports a result past the
    bound as too large rather than truncating it. **[New]**
13. <a id="f-13"></a>Events emitted by the public part of a privacy-preserving
    transaction are indexed as a public transaction's are
    (`lee/state_machine/src/validated_state_diff/mod.rs:586-621`), and execution
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

Every read in this section answers at the head or on the final chain, as the
caller chooses ([#96](#f-96)).

**`query_account(account_id) -> PointerResult<FfiOption<FfiAccount>, OperationStatus>`**

Returns the account record as it stands at the level read: its nonce, and one
actor state per program holding state on the account.

14. <a id="f-14"></a>The query returns the account's nonce and its actor states,
    each keyed by the account identifier of the program it belongs to
    (`lez/indexer/service/protocol/src/lib.rs:100-110`). The native balance is
    the actor state at the native token program, and the API documents how to
    read it. **[Ready]**
15. <a id="f-15"></a>An identifier the state holds no record for returns an
    absent `FfiOption`, distinct from a recorded account whose actor states are
    empty and from a backend failure. **[Ready]**

**`query_account_view(account_id, program_account_id, block_id) -> PointerResult<FfiOption<FfiAccount>, OperationStatus>`**

Returns an account's nonce and the actor state of one program, optionally pinned
to a block.

16. <a id="f-16"></a>The FFI exposes a read of one program's actor state on an
    account, with the account's nonce, at the tip or at a pinned block,
    exporting the existing `getAccountView` and `getAccountViewAtBlock` node RPC
    methods (`lez/indexer/service/rpc/src/lib.rs:83-91`). Nothing bounds how
    many actor states an account holds or how large each is, and any party can
    write its own onto any account
    (`lez/indexer/service/protocol/src/lib.rs:136-146`), so a whole-account read
    is not a safe way to read one balance; this is. **[Ready, not exposed]**

**`query_account_summary(account_id) -> PointerResult<FfiOption<FfiAccountSummary>, OperationStatus>`**

Returns an account's nonce, its native balance, and the size of each actor state
rather than its bytes.

17. <a id="f-17"></a>The FFI exposes an account summary: the nonce, the native
    balance, and the size of each actor state rather than its bytes, exporting
    the existing `getAccountSummary` node RPC method, so a caller learns which
    programs hold state on an account in a response whose size follows their
    count. **[Ready, not exposed]**
18. <a id="f-18"></a>An identifier the state holds no record for returns an
    absent `FfiOption`, on the same terms as the account read ([#15](#f-15)).
    The RPC method returns a default summary for it today
    (`lez/indexer/service/src/service.rs:151-161`), which a caller cannot tell
    from a recorded account that is empty. **[New]**

**`query_balances(account_id) -> PointerResult<FfiOption<FfiBalances>, OperationStatus>`**

Returns a public account's balances, decoded: its native balance, and its
balance in each token a builtin token program issues.

19. <a id="f-19"></a>The FFI returns a public account's native balance, decoded
    from its actor state at the native token program, as the summary already
    decodes it (`lez/indexer/service/src/service.rs:164`) and as the sequencer
    service's `getAccountBalance` serves it today
    (`lez/sequencer/service/rpc/src/lib.rs:82`). **[Ready, not exposed]**
20. <a id="f-20"></a>The same call returns the account's balance in each token a
    builtin token program issues, decoded, each with the identifier of the token
    it is denominated in, and the API documents which programs it decodes. A
    wrapped token's balance sits not on the holder's account but in a holding
    account the program derives from the token and the holder, whose data is the
    balance (`lez/programs/wrapped_token/core/src/lib.rs:171-198`), so the node
    resolves it for each wrapped token deployed. Solana's node does the same for
    a fixed set of well-known programs, SPL Token among them, and returns any
    other program's accounts as raw bytes
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.8](../appendix/blockchain-api-sdk-ecosystem.md#18-get-a-balance)).
    A balance held through any other program is likewise left to the caller,
    whose SDK decodes the actor state from the program's IDL. **[New]**

**`query_account_at_block(account_id, block_id) -> PointerResult<FfiOption<FfiAccount>, OperationStatus>`**

Returns the account record as it stood at a given block, rather than at the tip.

21. <a id="f-21"></a>The FFI exposes an account read pinned to a block
    identifier, returning the account record as it stood at that block, on the
    same terms as the read at the tip. This exports the existing
    `getAccountAtBlock` node RPC method
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[Ready, not exposed]**
22. <a id="f-22"></a>The pinned read returns the hash of the block it answered
    against, and the API documents that a block identifier at the head names a
    height rather than a block: until it is final, the same identifier can
    resolve to a different block after the head is replaced, and the hash is
    what tells a caller which one it read. **[New]**

**`query_accounts(account_ids, block_id) -> PointerResult<FfiVec<FfiOption<FfiAccount>>, OperationStatus>`**

Returns one result per requested account identifier, optionally pinned to a
block.

23. <a id="f-23"></a>The FFI exposes a batch account read taking a list of
    account identifiers and returning a result per identifier in request order.
    An identifier the state holds no record for is reported absent in its entry
    rather than failing the call. **[New]**
24. <a id="f-24"></a>The batch account read accepts an optional block identifier
    that pins every account in the batch to the same block, so a multi-account
    read is internally consistent under concurrent ingestion. **[New]**
25. <a id="f-25"></a>The batch read also returns, per account, the summary of
    [#17](#f-17) or the view of one program's actor state of [#16](#f-16) in
    place of the whole record, as the caller asks, so a batch never carries the
    unbounded actor states [#16](#f-16) warns of. The wallet reads its accounts'
    nonces in a batch today, through the sequencer service's `getAccountsNonces`
    (`lez/sequencer/service/rpc/src/lib.rs:98-102`). **[New]**

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

26. <a id="f-26"></a>The FFI returns, for a list of commitments and a list of
    message positions, a membership proof per entry in request order and the
    commitment set root those proofs are proven against. A commitment or
    position the set does not hold is reported as absent per entry rather than
    failing the call. This is the data the sequencer service's
    `getProofsAndRoot` returns today
    (`lez/sequencer/service/rpc/src/lib.rs:131-145`), which the wallet reaches
    the sequencer for (`lez/wallet/src/lib.rs:1310-1325`), and the state the
    node already holds answers it (`lee/state_machine/src/state/mod.rs:206`,
    `lee/state_machine/src/state/mod.rs:431-438`). **[Ready, not exposed]**
27. <a id="f-27"></a>The proofs and the root returned by one call are consistent
    with one another: every proof verifies against the returned root, as those
    `getProofsAndRoot` returns do. **[Ready, not exposed]**
28. <a id="f-28"></a>The API documents that this read names to the node the
    commitments and message positions the caller is about to spend, which is
    what a node shared by several wallets learns from it, and Privacy
    [#7](#priv-7)'s list of disclosing calls includes it. Soft [#6](#soft-6)
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

**`query_transaction_status(hash) -> PointerResult<FfiTransactionStatus, OperationStatus>`**

Returns how far a transaction has progressed, where it sits, and whether its
execution succeeded.

29. <a id="f-29"></a>The FFI exposes a per-transaction status query returning a
    level from a documented set that distinguishes at minimum the following, and
    each level documents what a caller may conclude from it, that documented
    meaning being what the API guarantees:

    - **not known**, on the terms of [#33](#f-33);
    - **received**: the transaction is in the pending set the node holds or
      observes ([#42](#f-42)), having been submitted through this node or seen
      on the LEZ peer-to-peer network;
    - **produced**, in sequencer mode only: in a block this node produced and
      has not yet seen inscribed ([#98](#f-98));
    - **inscribed**: in a LEZ block carried by an L1 block that is not yet
      final, which an L1 reorganisation or a competing inscription can still
      replace, returning the transaction to received or dropping it;
    - **finalised**: inscribed in an L1 block that is final, so a caller acting
      on it is acting on a settled result. This is not a statement about how
      much of the chain the node has processed;
    - **dropped**, on the terms of [#32](#f-32).

    The levels map onto `BedrockStatus`, whose `Pending`, `Safe`, and
    `Finalized` (`lez/common/src/block.rs:118-121`) the node sets only as
    `Pending` and `Finalized` today, and the mapping is documented. Before a
    transaction is finalised its level can move back as well as forward, and the
    API documents when. The node reads blocks only once final
    (`lez/chain_state/src/zone_indexer.rs:54-86`) and stamps every block it
    stores `Finalized` (`lez/indexer/core/src/block_store.rs:362`); the
    sequencer service already follows inscriptions at the L1 tip and folds them
    into its head (`lez/chain_state/src/chain.rs:1-3`). No level comes before
    inscription, since a node learns of a block only from its inscription
    ([Assumption A7](#a7-lez-blocks-are-not-propagated-over-the-lez-p2p-protocol)).
    Whether a node can also see an inscription while it is still in the L1
    mempool is open, and a level is added if it can. **[New]**

30. <a id="f-30"></a>The per-transaction status query returns the identifier and
    hash of the block containing the transaction and the transaction's index in
    it, which the node's store already resolves from the transaction hash
    (`lez/indexer/core/src/block_store.rs:161-163`) **[Ready, not exposed]**,
    and the L1 block the level was assessed against, so a caller can tell how
    current the answer is. The store already records the L1 slot of the
    validated tip (`lez/indexer/core/src/block_store.rs:211-216`) **\[Ready, not
    exposed\]**; the rest is **[New]**.

31. <a id="f-31"></a>The per-transaction status query reports execution success
    or failure, with a typed reason on failure, for any transaction the node has
    executed. A charged transaction whose action fails is included in its block
    and keeps its fee, and its receipt records the failure
    (`lez/common/src/block.rs:79-84`), which the FFI block type and
    `query_transaction_receipt` carry; that outcome is **[Ready]**. The reason
    is computed as free text when the action fails and then discarded
    (`lee/state_machine/src/validated_state_diff/mod.rs:66-72`), so a typed
    reason is **[Computed, not persisted]**.

32. <a id="f-32"></a>A transaction this node has seen, because it was submitted
    through this node or observed in its pending set, and that can no longer be
    included, is reported as dropped rather than as not known. A transaction can
    no longer be included once any of the conditions it must meet can never hold
    again: the nonce of an account it consumes has moved past the nonce it
    declares (`lez/indexer/service/protocol/src/lib.rs:297`); a nullifier it
    spends has been spent; or its block or timestamp validity window, which only
    a privacy-preserving transaction carries
    (`lez/indexer/service/protocol/src/lib.rs:397-398`), has passed
    (`lee/state_machine/src/validated_state_diff/mod.rs:358-381`). A transaction
    with an open window and no such conflict never becomes dropped on these
    terms, and [#43](#f-43) bounds how long the node keeps it. For the
    transactions it has seen, the node keeps what it needs to tell, and the API
    documents for how long. **[New]**

33. <a id="f-33"></a>The API documents that a transaction this node never saw,
    absent from the chain and from the pending set, is reported as not known,
    whether it was never submitted or was submitted elsewhere and dropped, no
    record of a declined transaction being retained. **[New]**

**`query_status() -> *mut c_char`**

Returns the node's own ingestion state as a JSON document: `state`,
`indexed_block_id`, `last_error`, `stall_reason`, `cross_zone_halt`, and
`cross_zone_peers`.

34. <a id="f-34"></a>The query reports the ingestion state as one of `Starting`,
    `Syncing`, `CaughtUp`, `Error`, `Stalled`, or `Halted`, together with the
    indexed block identifier, and carries the stall and cross-zone diagnostics
    when ingestion stopped. A state a client build does not recognise is treated
    as not known healthy rather than as a decode failure, and so is a peer
    health value in `cross_zone_peers` it does not recognise, such as
    `PinMismatch` (`lez/indexer/core/src/status.rs:64-78`), which the FFI's
    documented list omits. **[Ready]**
35. <a id="f-35"></a>The status query reports the node's mode: whether it is in
    sequencer mode and, if it is, whether it is accepting transactions or still
    starting, as the sequencer service's `getSequencerStatus` reports today
    (`lez/sequencer/service/rpc/src/lib.rs:158-159`); and, outside sequencer
    mode, whether the node has a peer to relay a submission through. A caller
    learns before submitting whether its transaction is admitted here or
    relayed, and whether it can be either. **[New]**
36. <a id="f-36"></a>The status query signals a null handle and a serialisation
    failure as distinguishable outcomes rather than returning a bare null
    pointer for both
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

37. <a id="f-37"></a>The FFI accepts a signed transaction and returns the
    transaction hash a caller then tracks through `query_transaction_status`. A
    node in sequencer mode admits the transaction to its own pending set; any
    other node relays it over the LEZ peer-to-peer network to the nodes that
    sequence, which is not part of this API. The node FFI has no such function.
    The sequencer service's FFI in `core_ffi` has one,
    `sequencer_ffi_send_transaction`
    (`lez/core_ffi/src/sequencer/api/query.rs:408-438`), which is prior art for
    the shape. **[New]**
38. <a id="f-38"></a>Every node validates a transaction before it admits or
    relays it, against its head state, as the sequencer service screens fees
    against its head today
    (`lez/sequencer/actors/executor/src/actor.rs:369-384`), and a transaction it
    declines is reported as a typed failure. The failures distinguish at least:
    a malformed transaction; one that fails a stateless check; one too large for
    a block; one declaring more gas than a block allows; one whose fee, or whose
    proof-committed reserve less its tip, is below the current price at the
    head; one whose payer cannot fund the fee or is not authorised to pay it
    (`lez/sequencer/core/src/fees.rs:13-33`,
    `lez/programs/fee/core/src/validity.rs:17-39`); and one invoking a program a
    user transaction may not call. These are reported separately from a node
    that cannot take a submission at all: one still starting, or, outside
    sequencer mode, one with no peer to relay through. A node in sequencer mode
    also reports a pending set that is full. The sequencer service's RPC gives
    three outcome codes of their own today, a full mempool, a privacy-preserving
    reserve below the head fee, and not yet online
    (`lez/sequencer/service/rpc/src/lib.rs:18-27`), and reports a public fee
    shortfall, an oversized transaction, and a call to a sequencer-only program
    as invalid parameters with text
    (`lez/sequencer/actors/rpc_server/src/actor/service.rs:58-68`,
    `lez/sequencer/actors/rpc_server/src/actor/service.rs:467-475`); the
    sequencer FFI reports every admission failure alike, apart from a
    transaction it cannot decode. A caller re-prices on an insufficient fee,
    funds the payer when it cannot pay, retries on a full pending set or a node
    that cannot take a submission yet, and does not retry the others. **[New]**
39. <a id="f-39"></a>Outside sequencer mode, acceptance means the node validated
    the transaction and handed it to the peer-to-peer network, not that a
    sequencer admitted it. Gossip forwards a transaction whatever a receiving
    sequencer's admission decides
    (`lez/sequencer/actors/gossip/src/actor.rs:528-540`), and a publication that
    cannot go out at once waits in a bounded queue that drops its oldest entry
    on overflow (`lez/sequencer/actors/gossip/src/actor.rs:66`,
    `lez/sequencer/actors/gossip/src/actor.rs:114`), so a sequencer's refusal
    never comes back to the relaying node, and acceptance may mean queued for
    relay. The API documents both, and the caller learns what became of the
    transaction from the status query ([#29](#f-29), [#32](#f-32)). **[New]**
40. <a id="f-40"></a>Submission neither signs nor modifies the transaction it is
    given. What is relayed is the transaction accepted, re-encoded at most, and
    the hash returned is the hash of that transaction. **[New]**

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
(`lez/sequencer/core/src/lib.rs:1793-1804`), so no drop is retrievable
afterwards and the pending set is the only surface on which its disappearance is
visible. It is the proof a wallet abandons, though, rather than status, that
makes the pending set a requirement here.

Which pending set a node can show depends on its mode. A sequencer's pending set
is a bounded mempool, fed both by submission and by gossip from its peers, and
it is only pushed to today (`lez/mempool/src/lib.rs:66-92`). A node outside
sequencer mode holds none, but the transactions sequencers gossip travel on a
peer-to-peer topic per zone (`lez/sequencer/actors/gossip/src/actor.rs:338-340`)
that any node following the zone can observe. Either way, one transaction sits
in several pending sets at once, none of them observable today, until it appears
in a block
([Appendix: Blockchain API and SDK Ecosystem, section 1.26](../appendix/blockchain-api-sdk-ecosystem.md#126-inspect-the-mempool-or-pending-set)).

**`query_pending_digests(cursor, limit) -> PointerResult<FfiVec<FfiPendingDigest>, OperationStatus>`**

Lists the pending set as digests: per entry, its transaction hash and the
account identifiers, nonces, and nullifiers it touches.

41. <a id="f-41"></a>The FFI lists pending transactions as digests carrying, per
    entry, its transaction hash together with the account identifiers, nonces,
    and nullifiers the transaction acts on, so a caller detects a conflict with
    a transaction it is about to build without fetching bodies. **[New]**
42. <a id="f-42"></a>In sequencer mode the listing is the pending set the node
    builds blocks from, together with the transactions of blocks it produced and
    has not yet seen inscribed, each entry marked as which. A block's
    transactions leave the mempool when it is produced
    (`lez/sequencer/core/src/lib.rs:1587-1591`), so without them a wallet's
    conflict check would miss the transactions most likely to conflict. Outside
    sequencer mode, the listing is the transactions the node has observed on the
    LEZ peer-to-peer network and not yet seen inscribed, together with those
    submitted through it. In either mode an entry another node has included
    leaves the listing once the inscription is seen. The API documents which a
    node serves, and that in neither mode does absence from the listing show
    that a transaction is not pending elsewhere. **[New]**
43. <a id="f-43"></a>Outside sequencer mode, the observed set is bounded, admits
    only a transaction that passes the node's validation ([#38](#f-38)), as the
    gossip actor evaluates each transaction it receives
    (`lez/sequencer/actors/gossip/src/actor/validation.rs:57-89`), and lets an
    entry go after a documented time, after which the status query reports it as
    no longer observed rather than as received. **[New]**
44. <a id="f-44"></a>The listing is bounded and paginated by cursor, on the same
    terms as the other paginated reads, and reports whether more entries remain.
    The pending set turns over as entries are included or dropped, so an offset
    into it names a different entry from one call to the next. **[New]**
45. <a id="f-45"></a>A page of the digest listing costs no more to serve than a
    page of blocks of the same limit, measured under Performance [#1](#perf-1),
    so a wallet re-checks for a conflict throughout a proof generation and
    abandons the proof rather than completing one it knows will be rejected.
    **[New]**
46. <a id="f-46"></a>The pending set is queryable by transaction hash, through
    `query_pending_entry` below, so a caller holding the hash
    `submit_transaction` returned asks whether that transaction is still pending
    without walking the listing. The listing also reports the pending set's size
    and, in sequencer mode, its capacity (`lez/mempool/src/lib.rs:30-32`), which
    a caller retrying on a full pending set reads. **[New]**

**`query_pending_entry(hash) -> PointerResult<FfiOption<FfiTransaction>, OperationStatus>`**

Returns one pending transaction in full, by its transaction hash.

47. <a id="f-47"></a>The FFI returns a pending transaction's body for a
    transaction hash, in the same shape `query_transaction` returns an included
    one, and reports absence distinctly from failure: an entry included or
    dropped since the caller last saw it is gone rather than an error. **[New]**

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

48. <a id="f-48"></a>The FFI exposes a subscription to finalised blocks that
    notifies a registered consumer of each block as it becomes final, exporting
    the existing `subscribeToFinalizedBlocks` node RPC method. The consumer is
    notified through a callback rather than by polling, and reads a block it is
    told of through `query_block_by_hash`. **[Ready, not exposed]**
49. <a id="f-49"></a>Each notification carries the block's height and its hash
    together, so a consumer knows which block occupied the position without a
    second read. The existing subscription yields a height alone
    (`lez/indexer/service/rpc/src/lib.rs:45`), which does not identify a block
    once the head can be replaced; the pair already exists as `BlockMeta`
    (`lez/common/src/block.rs:11-14`). **[New]**
50. <a id="f-50"></a>The FFI exposes a subscription to the head, notifying a
    consumer of each block as it enters the head, inscribed or, in sequencer
    mode, produced; of each head block that is replaced, with the block that
    replaces it; and of each head block as it becomes final. The sequencer
    service already folds each L1 update into its head as an extension or a
    conflict with the blocks it orphans
    (`lez/sequencer/core/src/lib.rs:350-358`). **[New]**
51. <a id="f-51"></a>A notification that cannot be delivered, because the
    consumer is not keeping up, ends the subscription with a final notice
    carrying the last position delivered, so the consumer resumes from it
    ([#52](#f-52)) rather than missing a block. The block subscription logs a
    failed send and goes on to the next block today
    (`lez/indexer/service/src/service.rs:526-534`). A resume position below the
    retention floor is answered as [#82](#f-82) defines. **[New]**
52. <a id="f-52"></a>The block subscription accepts a start position: the
    identifier and hash of the block the consumer last processed. Delivery
    resumes from the block after that position, so a consumer that reconnects
    observes no gap, and a position whose hash does not match the block the node
    holds at that identifier is reported as such rather than resumed from.
    Neither existing RPC subscription carries a position:
    `subscribeToFinalizedBlocks` takes no arguments, and `subscribeToEvents`
    filters by transaction, program, and selector but not by position
    ([Appendix: Blockchain API and SDK Ecosystem, section 1.33](../appendix/blockchain-api-sdk-ecosystem.md#133-resume-a-stream-from-a-known-position)).
    **[New]**

**`unsubscribe(subscription) -> OperationStatus`**

Cancels a registered subscription and releases the resources it holds.

53. <a id="f-53"></a>Cancelling a subscription stops delivery, and no callback
    fires after the call returns. **[New]**

**`subscribe_to_events(filter, from_block, callback, user_data) -> PointerResult<FfiSubscription, OperationStatus>`**

Registers a consumer that is called with each new event record matching a
filter, optionally resuming from a position the consumer already processed.

54. <a id="f-54"></a>The FFI exposes a subscription to program events, exporting
    the existing `subscribeToEvents` node RPC method
    (`lez/indexer/service/rpc/src/lib.rs:48-49`) with the filter it takes by
    transaction, program, and selector, and delivering each record in the shape
    `query_events` returns. Cancellation, and delivery to the caller that opened
    the subscription alone, are on the same terms as the block subscription.
    Today the subscription stops at the node RPC, so no application can follow a
    program's events. A filter the node's event filter does not persist is
    refused at subscription, as the RPC refuses it
    (`lez/indexer/service/src/service.rs:80-87`), rather than accepted and left
    silent. **[Ready, not exposed]**
55. <a id="f-55"></a>The event subscription resumes from a block identifier and
    hash on the same terms as the block subscription, [#52](#f-52), so a
    consumer following a program's events across a disconnection misses none. A
    resume position outside the node's event-filter history is answered as data
    the node does not hold, as [#12](#f-12) defines for the query. **[New]**

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

56. <a id="f-56"></a>The query returns at most `limit` transactions from the
    position the cursor names, or from the start of the walk when the cursor is
    absent, and stops early at the end of the account's history rather than
    failing. The bounded read exists; the cursor replaces the offset it takes
    today. **[New]**
57. <a id="f-57"></a>Each entry carries, beside the transaction, the identifier
    and hash of its block, its index in the block, and whether its action
    applied, so a deposit tracker judges each entry without a status call per
    entry. History entries are bare transactions today
    (`lez/indexer/ffi/src/api/types/transaction.rs:343-346`). **[New]**
58. <a id="f-58"></a>`query_transactions_by_account` accepts an ordering
    parameter supporting both oldest-first and newest-first. Newest-first is the
    order a deposit tracker reads in. **[New]**
59. <a id="f-59"></a>The page is walked by an opaque cursor rather than by a
    numeric offset, and the cursor remains stable across ingestion: a caller
    resuming from one neither skips nor repeats an entry that existed when the
    walk began, however many transactions have landed since. The per-account
    index is append-only (`lez/storage/src/indexer/write_atomic.rs:64-76`), so
    an offset from the oldest entry holds that property, but an offset from the
    newest, which is how a deposit tracker reads, shifts with every arrival, and
    so does any offset once history is pruned. The cursor is opaque to the
    caller, which may not construct one or infer a position from it. **[New]**

**`query_blocks(from, limit, order) -> PointerResult<FfiVec<FfiBlock>, OperationStatus>`**

Returns a page of blocks after `from`, `from` itself excluded, walked in the
requested direction. With `from` absent, a newest-first walk starts at the
indexed tip and an oldest-first walk at the retention floor. Exported today as
`query_block_vec`, which descends only.

60. <a id="f-60"></a>The query returns at most `limit` blocks, descending from
    `from`, or from the indexed tip when `from` is absent. **[Ready]**
61. <a id="f-61"></a>The bound is documented as exclusive in both directions.
    The store already implements it that way when descending, from
    `before_id.saturating_sub(1)`
    (`lez/storage/src/indexer/read_multiple.rs:11-12`) **[Ready]**; ascending is
    new with [#62](#f-62) **[New]**.
62. <a id="f-62"></a>`query_blocks` accepts an ordering parameter supporting
    both oldest-first and newest-first, on the same terms as
    `query_transactions_by_account`. Oldest-first is the order a consumer
    scanning forward reads in: a wallet syncing private accounts walks ascending
    from the last block it processed to the tip, decrypting each
    privacy-preserving transaction body against its own viewing key. Descending
    from the tip cannot serve that walk. An ascending walk with `from` absent
    starts at the retention floor. **[New]**
63. <a id="f-63"></a>Every paginated response reports whether more results
    remain, and carries the position a caller resumes from, so a caller
    distinguishes the end of a result set from a page that happens to be short.
    For the account history and the pending listing that position is an opaque
    cursor the caller does not construct ([#59](#f-59)); for the walks keyed by
    position, the blocks, publications, and encrypted notes, it is the block or
    commitment-set position the next page starts after. No paginated return type
    carries either signal today. **[New]**

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

Returns the software the node is running, so a caller knows which build answered
it
([Appendix: Blockchain API and SDK Ecosystem, section 1.1](../appendix/blockchain-api-sdk-ecosystem.md#11-get-node-version-and-software-identity)).
Every surveyed chain reports this and LEZ reports none: `web3_clientVersion` on
Ethereum, `getVersion` on Solana, `getnetworkinfo` on Bitcoin and Zcash.

64. <a id="f-64"></a>The FFI reports the version of the node serving the call,
    together with enough build identity to tell two builds of the same version
    apart. An integrator that hits a defect can then say which build produced
    it, and one that knows a build is bad can route around it. Neither the FFI's
    `query_status` nor the node RPC's `checkHealth`, which the FFI does not
    export, reports a version
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**
65. <a id="f-65"></a>The version changes only across restarts, so a caller may
    cache it for the life of a connection and use it to detect that the surface
    it holds a schema for has been replaced. **[New]**

**`query_block(block_id) -> PointerResult<FfiBlockOpt, OperationStatus>`**

Returns the block at a given identifier: its header, its full transaction body,
and its bedrock status.

66. <a id="f-66"></a>The query returns the stored block for a known identifier
    and an absent `FfiBlockOpt` for one the node does not hold, keeping the two
    distinct from a backend failure. **[Ready]**
67. <a id="f-67"></a>The returned header carries the block identifier, previous
    block hash, own hash, and timestamp
    (`lez/indexer/service/protocol/src/lib.rs:178-184`), and the body carries
    every transaction in the block. **[Ready]**
68. <a id="f-68"></a>The returned body carries each user transaction's receipt,
    so a consumer reading a block can tell a transaction that took effect from
    one that only paid its fee. The stored block, the node RPC, and the FFI
    block type already carry them
    (`lez/indexer/service/protocol/src/lib.rs:207-252`,
    `lez/indexer/ffi/src/api/types/block.rs:10-45`). **[Ready]**

**`query_block_by_hash(hash) -> PointerResult<FfiBlockOpt, OperationStatus>`**

Returns the same block record, resolved by block hash rather than by height. The
store already holds the hash as a secondary index onto the height
(`lez/storage/src/indexer/read_once.rs:67-70`).

69. <a id="f-69"></a>The query returns the same record `query_block` returns for
    the corresponding height, and an absent `FfiBlockOpt` for a hash no indexed
    block carries. **[Ready]**

**`query_last_block() -> LastBlockIdResult`**

Returns the identifier of the node's last finalised block, inline, with no
allocation to free.

70. <a id="f-70"></a>The query returns the last finalised block identifier, and
    reports an empty chain as an outcome distinct from an error. **[Ready]**

**`query_chain_tip() -> PointerResult<FfiChainTip, OperationStatus>`**

Returns the tip as one record rather than as a height the caller must then
resolve.

71. <a id="f-71"></a>The FFI exposes the chain tip as a single call returning,
    for the head and for the final chain, the tip block's height together with
    its hash and timestamp, so learning about the tip does not cost a second
    call and a caller knows which block holds the position. Below the FFI,
    `getLastFinalizedBlockId` returns a height alone; the chain state keeps a
    tip as a height and hash (`lez/chain_state/src/apply.rs:38-41`), and the
    timestamp comes from a second read. **[New]**

**`query_network_identity() -> PointerResult<FfiNetworkIdentity, OperationStatus>`**

Returns what the node is following: which zone, and which Logos Blockchain chain
that zone settles to.

72. <a id="f-72"></a>The call returns the zone identifier the node is following,
    so an application can confirm which zone it is connected to. The node's
    channel configuration already holds it
    (`lez/indexer/core/src/config.rs:37`). **[Ready, not exposed]**
73. <a id="f-73"></a>The same call returns the chain identifier of the Logos
    Blockchain the zone settles to. A zone identifier alone does not distinguish
    the same zone running against different L1 networks, which is the case an
    integrator connecting to the wrong network hits first, and the two are
    answered together so a caller cannot check one and assume the other.
    **[New]**
74. <a id="f-74"></a>The chain identifier is the one inscribed in the L1 genesis
    block as a Cryptarchia parameter, a bounded UTF-8 string such as
    `logos-chain-1`, and not a value the node is configured with independently:
    a configured string would agree with whatever an operator typed rather than
    with the chain the node is settling to. The Logos Blockchain node serves it,
    read off the genesis inscription, at `GET /chain/id` (`logos-blockchain`
    `nodes/api-common/src/paths.rs:11`), and the node already builds a client
    for its L1 node (`lez/indexer/core/src/lib.rs:112-115`), so the node reads
    it from there. **[New]**
75. <a id="f-75"></a>Where the chain identifier cannot be obtained, because the
    L1 node is unreachable, the call reports it as unavailable rather than
    omitting it or returning a placeholder, so a caller can tell an unidentified
    chain from an unasked question. **[New]**

**`query_chain_parameters() -> PointerResult<FfiChainParameters, OperationStatus>`**

Returns the protocol parameters a transaction is built against, with the block
they were read at.

76. <a id="f-76"></a>The FFI returns the chain parameters a caller needs before
    it builds and proves a transaction, at the level read ([#96](#f-96)): the
    largest transaction a block accepts, which is the channel's maximum block
    size (`lez/programs/sequencer_stake/core/src/lib.rs:238-252`,
    `lez/sequencer/core/src/lib.rs:2490`); the per-block execution and storage
    gas caps; the fixed execution gas a privacy-preserving transaction is
    charged for verification and the length its proof is charged at; the
    smallest surplus refunded rather than kept as tip; and the rate at which a
    base fee moves between blocks (`lez/programs/fee/core/src/market.rs:15-54`).
    A wallet that learns a transaction is too large only on submitting it has
    already spent minutes proving it. The channel parameters change by committee
    vote, so each value is reported with the block it was read at, and the chain
    identifier of [#73](#f-73) is returned with them. **[New]**

**`query_program_ids() -> PointerResult<FfiVec<FfiProgramEntry>, OperationStatus>`**

Returns the programs deployed on the chain, each with its account identifier,
its image identifier, and, for a builtin, its name, together with the identifier
of the privacy-preserving circuit the chain verifies proofs against.

77. <a id="f-77"></a>The FFI exposes the programs deployed on the chain as its
    state records them, programs deployed after genesis included. A program is
    an account written through the program loader, and the state resolves an
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
78. <a id="f-78"></a>The same call returns the identifier of the
    privacy-preserving circuit whose proofs this node verifies, which a wallet
    compares with its own before it proves: a mismatch otherwise surfaces only
    when a transaction that took minutes to prove is rejected. That identifier
    is the one entry `getProgramIds` returns today, and the wallet's health
    check compares it with its own (`lez/wallet/src/cli/mod.rs:300-309`).
    **[Ready, not exposed]**

**`query_fee_state() -> PointerResult<FfiFeeState, OperationStatus>`**

Returns the fee market a transaction is priced against: the base fees at a
stated block, the band the next block's may move within, and the per-block gas
caps
([Appendix: Blockchain API and SDK Ecosystem, section 1.17](../appendix/blockchain-api-sdk-ecosystem.md#117-estimate-the-fee-or-fee-rate)).

79. <a id="f-79"></a>The FFI exposes the execution and storage base fees, the
    block height they were read at, and the per-block execution and storage gas
    caps, from the fee state the chain state holds
    (`lez/chain_state/src/apply.rs:335-344`). The wallet prices every fee
    declaration and every in-proof reserve against these values, read today from
    the sequencer service's `getFeeState`
    (`lez/sequencer/service/rpc/src/lib.rs:57-58`), and the node FFI serves none
    of them. **[Ready, not exposed]**
80. <a id="f-80"></a>The same call returns the lowest and highest base fees the
    next block can open at, in each dimension, as `getFeeState` does, so a
    caller sets a cap on the fee from the band rather than from a guess. The
    band follows from the fee state and the fee market's constants
    (`lez/sequencer/core/src/fees.rs:89-123`). The API documents that the band
    bounds the next block and is not an estimate for a given transaction.
    **[Ready, not exposed]**

The fee state is read at the level the caller chooses ([#96](#f-96)), and a
wallet pricing a transaction reads it at the head, as the sequencer service
quotes it today. A base fee can move by up to an eighth between blocks
(`lez/programs/fee/core/src/market.rs:16`), so a band read on the final chain,
which on testnet trails the head by about an hour, bounds only the block after
the final tip, and the API says so.

No tip history is required. A sequencer takes transactions from its pending set
in arrival order (`lez/mempool/src/lib.rs:40-58`,
`lez/sequencer/core/src/lib.rs:1591`), so at the commit read the tip, paid to
the block's producer whichever way a transaction pays, does not buy earlier
inclusion. Each receipt records the tip a transaction paid ([#68](#f-68)). For a
proof-committed fee that figure includes any surplus kept rather than refunded,
so it can exceed the tip the sender chose, which the transaction body carries.
Soft [#5](#soft-5) covers the case where inclusion comes to depend on the tip.

**`query_retention_floor() -> PointerResult<FfiRetentionFloor, OperationStatus>`**

Returns the earliest block the node can still answer for, separately for account
state pinned to a block and for transaction retrieval.

81. <a id="f-81"></a>The FFI exposes the node's retention floor: the earliest
    block for which account state can be read at a pinned block identifier, and
    the earliest block for which a transaction can be retrieved. A deployment
    that retains everything reports genesis and one bounded by Reliability
    [#7](#rel-7) reports a moving value, and a caller learns which by asking
    rather than by discovering it through a failed read. Where a deployment
    keeps some data further back than the rest, as Soft [#4](#soft-4) allows,
    the call reports retention per scope rather than one floor that is wrong for
    part of the data. **[New]**
82. <a id="f-82"></a>A read for data below the retention floor is answered as
    such, distinctly from a read for data that never existed. Asking for a block
    the node discarded and asking for one the chain never held are different
    questions with different answers: the first is retried against an archival
    source and the second is not. A caller learns which without a second call to
    the retention floor, which may itself have moved between the two. **[New]**

##### The node on the peer-to-peer network

What a caller can learn about the node's place in the LEZ peer-to-peer network.
The protocol between nodes is not part of this API
([Out of Scope](#out-of-scope)), but a node's connectivity is: a node with no
peers relays nothing and observes no pending transaction.

- **Ethereum**: `net_peerCount` and `net_listening` for any caller; Geth's
  `admin_peers` and `admin_nodeInfo` for the operator.
- **Bitcoin**: `getconnectioncount`, `getpeerinfo`, and `getnetworkinfo`.
- **Cosmos**: CometBFT's `/net_info`.
- **Solana**: `getClusterNodes`.

**`query_peer_count() -> PointerResult<FfiPeerCount, OperationStatus>`**

Returns how many LEZ peers the node is connected to.

83. <a id="f-83"></a>The FFI reports how many LEZ peers the node is connected
    to, inbound and outbound, and how many of them are in sequencer mode where a
    peer makes that known, so a caller can tell a node that can relay and
    observe from one that cannot. It is in the user category. The sequencer's
    gossip actor already tracks the peers it is connected to
    (`lez/sequencer/actors/gossip/src/actor.rs:82-86`). **[New]**

**`query_peers() -> PointerResult<FfiPeerInfo, OperationStatus>`**

Returns the node's own peer identity and listening addresses, and each connected
peer.

84. <a id="f-84"></a>The FFI returns the node's peer identifier and listening
    addresses, and, for each connected peer, its identifier, address, direction,
    the time it connected, and, where known, whether it is in sequencer mode. A
    peer's address is another operator's network identity, so the function is in
    the operations category ([#104](#f-104)), as Geth keeps `admin_peers` off
    its public endpoints. **[New]**

##### Private message delivery

Reading the messages a private account receives. A send to a private account is
published rather than applied, at a position in the commitment set: as a clear
publication whose envelope is in plaintext, always for a public-to-private send
and at the sender's choice for a private one, or as a sealed publication. The
receiving wallet reads the publications, finds those addressed to it, and later
consumes them. Among the surveyed chains only Zcash, whose shielded payments a
recipient finds by scanning, has a comparable structure. The sequencer service
serves it today (`lez/sequencer/service/rpc/src/lib.rs:112-129`), and the node
holds the data in its own state (`lee/state_machine/src/state/mod.rs:357-406`).

**`query_publications(from_position, limit) -> PointerResult<FfiPublicationPage, OperationStatus>`**

Returns a page of publications in commitment-set order, from a given position.

85. <a id="f-85"></a>The FFI lists publications in position order from a given
    position, each with its position: for a clear publication its plaintext
    envelope, and for a sealed one its commitment and encrypted note. Each page
    also carries, once, the recovery note bound to each destination of a clear
    publication in it, as the state already pages them
    (`lee/state_machine/src/state/mod.rs:178-181`,
    `lee/state_machine/src/state/mod.rs:383-406`) and the sequencer service's
    `getPublications` returns them. **[Ready, not exposed]**
86. <a id="f-86"></a>The listing is bounded and reports whether more entries
    remain, on the same terms as the other paginated reads, and takes no account
    identifier, so a wallet receiving messages discloses nothing beyond the
    position it resumes from. **[New]**

**`query_recovery_binding(address) -> PointerResult<FfiOption<FfiEncryptedNote>, OperationStatus>`**

Returns the encrypted note bound to a private address, if one is.

87. <a id="f-87"></a>The FFI returns the recovery note bound to a private
    address, or an absent `FfiOption` when none is, at the level read
    ([#96](#f-96)), exporting what the sequencer service's `getRecoveryBinding`
    serves today from its head. A wallet building a transaction reads it at the
    head: it checks whether each address it would bind is bound already, a
    duplicate binding making the transaction invalid, and whether the
    destination of a clear send is bound, a clear send to an unbound one being
    refused (`lez/wallet/src/lib.rs:1474-1520`). **[Ready, not exposed]**
88. <a id="f-88"></a>The API documents that this lookup names a private address
    to the node: the caller's own, or, when a sender checks a destination before
    a clear send, the address of the account it is about to pay. A consumer
    reading its own notes without disclosing its addresses reads them from the
    publication listing or the viewing key walk instead, neither of which takes
    an address. **[New]**

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

**`query_private_notes(from_block, from_hash, limit) -> PointerResult<FfiNotePage, OperationStatus>`**

Returns, block by block in ascending order, every encrypted note the chain
carries with the public values needed to decrypt and check it.

**`subscribe_to_private_notes(from_block, from_hash, callback, user_data) -> PointerResult<FfiSubscription, OperationStatus>`**

Delivers the same walk as new blocks arrive.

89. <a id="f-89"></a>The FFI exposes a walk, ascending by block from a given
    block, returning for each privacy-preserving transaction its hash, its index
    in the block, and whether its action applied, and for each encrypted note it
    carries the values a scanner needs to open and check it: for a private
    action, including the fee fallback's, its nullifier, its commitment with the
    commitment's position in the commitment set, and its encrypted post-state;
    for a sealed async message, its commitment, note, and position; and for a
    recovery binding, its address and note. Proofs, witness sets, and public
    bodies are omitted; a clear publication, whose envelope is plaintext, is
    read from the publication listing, [#85](#f-85). The notes are in the stored
    transaction (`lez/indexer/service/protocol/src/lib.rs:362-441`), whether an
    action applied is in the block's receipts, and the positions are in the
    state's commitment set and publications, so the work is a compact projection
    of data the node holds. **[New]**
90. <a id="f-90"></a>The walk covers every encrypted note whatever kind of
    private account it addresses: a regular private account, a private PDA, and
    a shared private account keyed from a group master secret, of either kind.
    The node needs no knowledge of which kind a note belongs to, and the walk
    does not depend on it. **[New]**
91. <a id="f-91"></a>The walk takes no viewing key, account identifier, or other
    value tied to the consumer's accounts. Every caller asking for the same
    range receives the same result, so the walk discloses to the node the range
    read and nothing else. **[New]**
92. <a id="f-92"></a>The walk resumes from a block identifier and hash on the
    same terms as the block subscription, [#52](#f-52), and is also delivered as
    a subscription, so a viewing key consumer following the tip neither polls
    nor misses a block. **[New]**
93. <a id="f-93"></a>The API documents what the walk serves for each kind of
    note, and that it serves every encrypted note in the range, whichever
    account it addresses. What a viewing key holder can and cannot establish
    from it is documented with the LEZ wallet API, which does the scanning.
    **[New]**

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

94. <a id="f-94"></a>The LEZ node module publishes its contract in the Logos
    Core contract format, covering every method and event the module exposes:
    its parameters, its return type, its error values, the category it belongs
    to ([#104](#f-104)), and the node modes it is available in ([#98](#f-98)).
    That format is a CDDL schema in the target and LIDL today, which the module
    wrapping the FFI already publishes, derived from its header. Where the
    format carries no field for the category or the mode per method, the
    contract expresses them another way the documentation states, such as
    separate user and administrative interfaces. The module describes its
    contract at run time, and each release documents which FFI function serves
    each module method. The existing `getSchema` describes the block, event
    record, and event filter types rather than the surface, and is not exported
    ([Appendix: Logos API Surfaces, section 2](../appendix/logos-api-surfaces.md#2-lez-indexer-rpc)).
    **[New]**

Every function defined above is further bound by the following.

095. <a id="f-95"></a>Every FFI function that takes the node handle returned by
     `start_node`, as its first argument the signatures above elide, signals a
     null handle as `OperationStatus::NullPointer` rather than as a not-found
     result. Every query exported today does so except `query_status`, which
     [#36](#f-36) addresses. **[Ready]**
096. <a id="f-96"></a>Every read of chain state takes a level, head or final,
     and answers against it. The head is every block the node has seen
     inscribed, final or not, and, in sequencer mode, the blocks it produced and
     has not yet seen inscribed; the final chain is the blocks inscribed in L1
     blocks that are final. A caller that omits the level reads the head, as the
     sequencer service answers today
     (`lez/sequencer/actors/executor/src/actor.rs:542-553`,
     `lez/sequencer/core/src/lib.rs:1956`). Each answer names the level and the
     block, with its hash, it was served at, because an answer at the head can
     be overtaken when the head is replaced. **[New]**
097. <a id="f-97"></a>Every read reports the tips it was served against: the
     identifier and hash of the last block at the head and on the final chain
     when it answered. A caller detects a stalled or lagging node from any
     response, not only from the status query. No return type carries them
     today. **[New]**
098. <a id="f-98"></a>Every function is available on every node unless the
     contract ([#94](#f-94)) marks it, or one of its outcomes, for sequencer
     mode. A node outside sequencer mode answers such a call as not available in
     its mode, an outcome distinct from not found, invalid argument, backend
     failure, and refusal ([#100](#f-100), [#108](#f-108)), so a caller learns
     to call a node in sequencer mode rather than retrying this one. **[New]**
099. <a id="f-99"></a>Every function above is exposed through the node module
     with the same semantics, including the not-found and error distinction
     required by Functionality [#100](#f-100). No capability reaching the FFI
     stops at the module boundary. The module that wraps the FFI today carries
     eight of its ten queries, omitting `query_events` and
     `query_transaction_receipt`, and is built against the LEZ `v0.3.1` FFI
     rather than the testnet 0.4 one. **[New]**
100. <a id="f-100"></a>The FFI and the module signal not-found,
     invalid-argument, and backend failure as three distinguishable outcomes on
     every query. The module currently flattens not-found and failure into an
     empty string
     ([Appendix: Logos API Surfaces, section 1](../appendix/logos-api-surfaces.md#1-lez-indexer-ffi)).
     **[New]**
101. <a id="f-101"></a>Errors are machine-readable and stable: a caller branches
     on a value rather than on message text, and that value does not change
     meaning between releases. Which values exist is for the implementation to
     decide; what this RFP requires is that two failures a caller would recover
     from differently never arrive as the same value. The node RPC returns the
     stock JSON-RPC `InternalError` code with free text for most failures,
     typing only an invalid event range, and the FFI's status type has five
     values, so most failures look alike
     ([Appendix: Blockchain API and SDK Ecosystem, section 1.34](../appendix/blockchain-api-sdk-ecosystem.md#134-structured-errors-and-a-code-taxonomy)).
     **[New]**
102. <a id="f-102"></a>The distinctions a caller acts on are, at minimum: a
     request that was malformed, which the caller fixes; a request for data
     below the retention floor, which the caller sends elsewhere; a request the
     node could not serve because it is stalled or lagging, which the caller
     retries; and a backend failure, which the caller retries with backoff. A
     transient failure is distinguishable from a permanent one without parsing
     text. **[New]**
103. <a id="f-103"></a>Every heap-allocating return has a documented matching
     free function, and calling it releases every allocation the return holds.
     The returns exported today have theirs **[Ready]**; the return types this
     RFP adds need theirs **[New]**.

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

**`start_node(runtime, config_path, storage_dir) -> PointerResult<NodeHandle, OperationStatus>`**

Starts the node against a configuration and a storage directory and returns the
handle every other function takes. It is the one function that takes no handle.

**`stop_node() -> OperationStatus`**

Stops the node and releases the handle.

**`reset_storage(config_path)`**

Stops the node and deletes the store of the zone the configuration names,
without restarting it. Exposed by the node module only.

104. <a id="f-104"></a>Every function this RFP defines belongs to exactly one of
     the three categories of [Function categories](#function-categories), and
     the contract of [#94](#f-94) states which. The user category holds every
     function defined under the headings above but `query_peers`. The operations
     category holds `query_peers` ([#84](#f-84)), and any function the module
     exposes that reports how the node runs in more detail than its status does,
     such as metrics. The administrative category holds `start_node`,
     `stop_node`, and `reset_storage`, together with any other function the
     module exposes that changes how the node runs, such as the FFI's
     `init_logger` were the module to expose it. A function added to the surface
     later is assigned to a category when it is added. **[New]**
105. <a id="f-105"></a>Each function exposed through the node module is
     individually grantable. Where the deployer's Logos Core access policy
     grants a caller a list of methods on the module, that caller reaches those
     methods and no others, and the module is built so that such a grant is
     enforced against it rather than refused as unenforceable. **[New]**
106. <a id="f-106"></a>An operations or administrative function refuses a call
     unless the caller is the host runtime, an operator of that runtime, or a
     caller the node's own configuration names for that category. The module
     decides this itself, from the caller identity Logos Core gives it: it
     cannot see which grant admitted a call, since grants are enforced before
     the module runs and the caller document carries only a kind, a name, and an
     instance (`logos-protocol` `cpp/logos_module_impl.h:223-283`). The refusal
     holds when no access policy is in force, when the policy leaves the module
     open to every caller, and when the runtime cannot identify the caller.
     Logos Core emits host, module, and unknown callers today, and specifies an
     operator kind it does not yet emit, so until it does the documentation
     states which callers can start, stop, and reset the node, and a node can be
     configured to start on load. **[New]**
107. <a id="f-107"></a>A user function requires nothing of a caller beyond
     admission to the module. No user function requires a caller to register, to
     name a wallet, or to have made an earlier call, and the node holds no state
     for a caller other than the subscriptions that caller opened. A cursor
     carries the position it resumes from, so a walk part way through holds
     nothing on the node. **[New]**
108. <a id="f-108"></a>A refused call is reported as an outcome distinct from
     the not-found, invalid-argument, and backend failure outcomes of
     [#100](#f-100), and has no effect on the node. A caller does not retry it.
     **[New]**
109. <a id="f-109"></a>Calls from different callers are served concurrently, up
     to a documented bound on calls in flight, and a slow call from one caller
     delays another's only once that bound is reached. The module runs with
     Logos Core's concurrent dispatch, where a module dispatches one call at a
     time by default (`logos-module-builder` `docs/configuration.md`,
     `concurrency`), and the module wrapping the FFI sets nothing today. A
     cursor issued to one caller is neither advanced nor invalidated by another
     caller's calls. **[New]**
110. <a id="f-110"></a>A subscription belongs to the caller that opened it, and
     is named by a handle no other caller can guess, so callers the runtime
     cannot tell apart, which arrive as one unknown caller, still cannot cancel
     or read one another's subscriptions. `unsubscribe` cancels only a
     subscription whose handle the caller holds, a caller's subscriptions end
     when it disconnects, and one caller's cancellation, disconnection, or
     failing callback neither stops nor delays delivery to any other caller.
     **[New]**
111. <a id="f-111"></a>Once the node is stopped, every call reaching the module
     from every caller reports the node as stopped, distinctly from a backend
     failure, and each open subscription receives a final notification that
     delivery has ended. The FFI handle is released on stop, so this is the
     module's obligation rather than the FFI's. **[New]**

#### Usability

1. <a id="use-1"></a>Provide the LEZ node module, covering every function in
   Functionality, usable to build Logos modules for reading LEZ state, following
   the chain, and tracking deposits.
2. <a id="use-2"></a>Provide a CLI that covers core functionality: read an
   account at current and at a pinned block, read a batch of accounts, read a
   block, read a transaction, read a transaction's effects, receipt, and status,
   read an account's balances, list an account's transactions with ordering and
   pagination, follow the head and the final chain, follow a program's events,
   read the node's status, mode, and peer count, read the chain parameters and
   the fee market, list the pending set, list publications, and walk the
   encrypted notes of a block range. Every read takes the level of [#96](#f-96).
   The CLI may have fewer features than the FFI but must support all essential
   operations.
3. <a id="use-3"></a>Provide a worked deposit-tracking example, in the CLI or as
   a documented reference consumer, that follows the chain from a chosen start
   block, detects credits to a supplied set of public accounts, and reports each
   credit with its transaction hash, amount, block, and status level.
4. <a id="use-4"></a>Document which state is readable and which is not for
   privacy-preserving transactions, including that a send published clear, from
   a public or a private sender, reveals its amount, its receiving private
   address, and its sender as presented, but not who controls that address, and
   that what a private account holds is not trackable from the node's data
   without its viewing key.
5. <a id="use-5"></a>Return clear, actionable error messages for every failure
   mode, each carrying the machine-readable value required by Functionality
   [#101](#f-101) alongside the text.
6. <a id="use-6"></a>Document the semantics of every pagination parameter,
   including the exclusivity of the block bound, what a cursor guarantees across
   ingestion, and the behaviour when new data lands during a walk.
7. <a id="use-7"></a>Provide a reference Logos Core access policy for each kind
   of deployment in
   [Who uses it, and how it is deployed](#who-uses-it-and-how-it-is-deployed): a
   user's own node, on the same machine as the wallet or another, a node in an
   integrator's infrastructure serving its users' wallets, and a node an
   operator offers as a service. Each grants the user category to wallets, the
   operations category to the operator's monitoring, and the administrative
   category to no caller beyond those Functionality [#106](#f-106) admits.
8. <a id="use-8"></a>Document how a deployer grants a caller a single function
   or a subset of the user category, such as the reads without
   `submit_transaction`.
9. <a id="use-9"></a>Document the head and the final chain for a caller: what
   each level guarantees, how long a block takes to become final on testnet, how
   a read at the head reports that the head was replaced, and which level a
   deposit tracker, a wallet building a transaction, and an explorer should each
   read at.

#### Reliability

1. <a id="rel-1"></a>A pinned account read at a given block identifier returns
   the same value on every call for as long as that block is the one at the
   identifier, which on the final chain is for good, and returns the hash of the
   block it answered against so a caller can tell that a repeated read resolved
   to the same block.
2. <a id="rel-2"></a>A batch account read pinned to a block identifier returns a
   set of accounts consistent with one another as at that block, not a mixture
   of values read at different points during concurrent ingestion.
3. <a id="rel-3"></a>Pagination remains valid across ingestion: a caller walking
   an account's transactions by cursor, or blocks, publications, and encrypted
   notes by position, neither skips nor repeats an entry that existed when the
   walk began.
4. <a id="rel-4"></a>A subscription consumer that disconnects and reconnects
   with its last processed block identifier and hash receives every block after
   that position, with no gap and no assumption that the consumer was connected.
5. <a id="rel-5"></a>A stalled or lagging node is detectable from any read,
   through the tips every response carries (Functionality [#97](#f-97)).
6. <a id="rel-6"></a>A stalled node is reported as stalled by the status surface
   rather than serving stale reads silently.
7. <a id="rel-7"></a>The growth of history is bounded by something the
   deployment controls rather than by the length of the chain. The node runs
   both as infrastructure an operator provisions and inside Basecamp on an end
   user's machine, where no one is watching a disk and no one can add one, so
   growth that a server operator would size for is a defect on a desktop. A
   deployment can put a ceiling on the history the node retains, blocks,
   transaction bodies, effects, per-account indexes, and events, and reaching
   that ceiling degrades what the retention floor reports rather than failing
   reads or exhausting the disk. The state the node validates against cannot be
   bounded this way, so the node reports its size and the documentation projects
   its growth.
8. <a id="rel-8"></a>No caller degrades the node for the others through the user
   set. Every user function bounds what one call consumes, including the size of
   a batch, the limit of a page, the span of a range, the size of a response,
   and the number of subscriptions one caller holds open. A call past a bound
   returns `InvalidArgument`, and the bounds are documented. A whole-account
   read (Functionality [#14](#f-14), [#23](#f-23)) of an account whose actor
   states exceed the response bound reports the account as too large to read
   whole, and the caller reads it one program at a time (Functionality
   [#16](#f-16)).

#### Performance

1. <a id="perf-1"></a>Document the latency of every exported function, measured
   rather than estimated, against a stated chain size and transaction density.
2. <a id="perf-2"></a>Where a function's cost varies with something the caller
   controls or observes, document what it varies with and report the worst case
   as well as the typical one. A read of a past block, a batch sized by the
   caller, and a page bounded by a limit each cost more than a read at the tip,
   and a caller choosing between them needs to know by how much.
3. <a id="perf-3"></a>A batch account read of N accounts costs materially less
   than N single reads, and the improvement is measured and documented for
   representative N.
4. <a id="perf-4"></a>Document the storage cost of persisting per-transaction
   effects: bytes per transaction, and projected growth against a stated block
   rate and transaction density.
5. <a id="perf-5"></a>Benchmarks are reproducible from the test suite and run
   against a LEZ devnet or testnet node.
6. <a id="perf-6"></a>Document the latency of the user category under concurrent
   callers, measured against a stated number of callers and open subscriptions,
   alongside the single caller figures.
7. <a id="perf-7"></a>Document the size of the viewing key walk against the full
   blocks it is drawn from, in bytes per block and per encrypted note, and the
   time the node takes to serve a stated number of blocks of it, so a consumer
   restoring from a viewing key knows what fetching the walk from genesis costs.
8. <a id="perf-8"></a>Document how one node scales with the deployments it
   serves, from a user's own node to one an integrator or an operator runs for
   many users: the callers, open subscriptions, and calls per second it sustains
   on stated hardware, and how its cost grows with the number of accounts its
   callers read and follow. Where a figure falls short of a node serving many
   users, the documentation says so.

#### Supportability

This RFP states its requirements on the node FFI, and the LEZ node module is
what exposes them to Logos Core under its contract: the module links the FFI
library and includes its generated header, so the two ship together and a
function reaches an application only when both carry it. "The FFI and module"
below means the node's FFI crate and the module that links it.

01. <a id="sup-1"></a>The FFI and module are delivered for LEZ testnet 0.4: both
    are built and tested against a LEZ devnet and against LEZ testnet 0.4.
02. <a id="sup-2"></a>End-to-end integration tests run in CI against a local
    stack of a Logos Blockchain node and LEZ nodes, at least one in sequencer
    mode and one outside it, with the node module loaded in a Logos Core runtime
    and called through it. The repository's `docker-compose.yml` assembles a dev
    Bedrock node, a sequencer committee, and the node's RPC service today,
    without the FFI, the module, or a Logos Core runtime, and LEZ's own CI runs
    a Bedrock node with its test fixtures rather than the compose file. The
    tests of the module and its access control drive a Logos Core runtime with
    several callers, as `logos-capability-module` drives its module through
    `logosctl` in doctests. Every function and outcome whose behaviour differs
    by mode is tested in each mode, including a call that is not available in a
    node's mode (Functionality [#98](#f-98)).
03. <a id="sup-3"></a>CI builds and tests the FFI and the module, and must be
    green on the default branch of every repository the work lands in. The
    module that wraps the FFI today has no build or test workflow.
04. <a id="sup-4"></a>Every hard requirement in Functionality, Usability,
    Reliability, and Performance has at least one corresponding test.
05. <a id="sup-5"></a>Tests cover, at minimum: a read of an identifier the state
    holds no record for; a pinned read at a block before and after a state
    change; a batch read spanning recorded and unrecorded accounts; effects for
    a public transaction, for the public leg of a privacy-preserving
    transaction, and for a transaction whose action reverted; membership proofs
    for a commitment and a message position the set holds and ones it does not;
    a clear publication from a public and from a private sender, and a sealed
    one, in the listing; the viewing key walk over a range, returning every
    encrypted note the range's transactions carry, for a regular private
    account, a private PDA, and a shared private account; a read at the head and
    on the final chain while a block is inscribed but not final, and a read at
    the head across the head being replaced; a transaction status at each
    documented level, a dropped public and a dropped privacy-preserving
    transaction among them; a submission to a node in sequencer mode and to one
    outside it, and to one with no peers; a subscription resumed from a stored
    position across a disconnection, and one resumed from a position whose hash
    does not match; an administrative call and an operations call from a caller
    holding no grant, with no access policy in force and with a policy that
    leaves the module open; and two callers holding subscriptions when one of
    them cancels.
06. <a id="sup-6"></a>A README documents end-to-end usage: building the FFI and
    the module that links it, the configuration and storage directory
    `start_node` requires, and step-by-step instructions for every operation via
    the CLI.
07. <a id="sup-7"></a>Provide documentation in repo for the FFI and module,
    covering the developer integration journey for reading state, tracking
    deposits, and confirming transactions.
08. <a id="sup-8"></a>Provide documentation in repo for the CLI, covering the
    core operator journey.
09. <a id="sup-9"></a>Provide a get started guide that takes a developer from an
    empty repository to a working Logos Core module calling the LEZ node module,
    covering the client generated from the module's contract, reading LEZ state
    through it, following the chain, and running the result against a devnet,
    with the node in the same runtime and reached through peering. A module
    calls another through its contract rather than linking its library, so the
    guide does not link the node FFI library. The README required above
    documents building and operating what this RFP specifies; this documents
    building something else on top of it.
10. <a id="sup-10"></a>Provide a full API reference for the node API, published
    per version as the Logos Storage module publishes its
    [API reference](https://logos-co.github.io/logos-storage-module/latest/api_reference.html),
    covering every module method and event, its parameters, its return type, its
    error values, its set, and the node modes it is available in. The reference
    is generated from the module contract required by Functionality [#94](#f-94)
    rather than maintained by hand, so it cannot drift from the surface it
    describes.
11. <a id="sup-11"></a>Provide a design decision log in repo recording each
    major design decision and choice: the context, the options considered, the
    option chosen, and the rationale and trade-offs. The log is updated as
    decisions are made and is current at each milestone delivery.
12. <a id="sup-12"></a>Provide a plan, agreed with the Logos Core maintainers
    and recorded in the design decision log, for how an application uses several
    zones while Logos Core runs one instance of each kind of module: whether
    through several instances of the node module, one per zone, or through one
    node serving several zones, and what an application sees, and for how long,
    when it moves between zones.

#### + Privacy

1. <a id="priv-1"></a>No function added by this RFP returns private state in the
   clear, or a viewing key, or any value from which either can be recovered
   without the viewing key, and no function accepts a viewing key, a nullifier
   key, or a decrypted note. Private post-states remain readable only by the
   viewing key holder, who decrypts them on its own side.
2. <a id="priv-2"></a>The effects and status surfaces do not report, for a
   privacy-preserving transaction, any linkage between its private inputs and
   its private outputs beyond the commitments and nullifiers already published
   on-chain.
3. <a id="priv-3"></a>Provide a privacy properties document covering: what a
   third party reading the node can learn about a public transaction, about the
   public leg of a privacy-preserving transaction, including the address its fee
   was paid from under either way of paying it, and about its private leg; what
   a send published clear reveals, from a public and from a private sender;
   which of those boundaries are enforced by encryption and which by the API
   declining to expose data it holds; and what an operator running their own
   node can see that a remote caller cannot. What a viewing key holder can learn
   is documented with the LEZ wallet API.
4. <a id="priv-4"></a>No function discloses one caller's activity to another.
   Which accounts, commitments, and transactions a caller asked about, which
   subscriptions it holds and with what filter, and which caller submitted a
   transaction are not readable by any other caller through this surface.
5. <a id="priv-5"></a>The node keeps no record tying a caller identity to what
   that caller read or submitted beyond what serving the call requires. Any
   logging that does so is off by default. This binds the node; what Logos Core
   records of each access decision is covered under Privacy [#6](#priv-6).
6. <a id="priv-6"></a>The privacy properties document states what the operator
   of a node shared by several wallets learns about each of them from its calls:
   the accounts and commitments it reads, the transactions it submits, and the
   caller identity those arrive under. It also states what Logos Core records
   about calls to the node, its draft specification giving every access decision
   an audit record.
7. <a id="priv-7"></a>A viewing key consumer can follow its accounts through
   calls that name none of them. The viewing key walk, the publication listing,
   and the subscription delivering the walk take positions only, and the
   documentation identifies every user function that takes an account
   identifier, a commitment, or a private address, so a consumer knows which
   calls disclose what it holds.

### Soft Requirements

If possible.

#### Functionality

1. <a id="soft-1"></a>A subscription filtered to a supplied set of accounts,
   delivering only blocks containing a transaction that touches one of them.
   Most surveyed chains that offer a block subscription also offer a filtered
   one; Bitcoin and Zcash push unfiltered streams only
   ([Appendix: Blockchain API and SDK Ecosystem, section 1.32](../appendix/blockchain-api-sdk-ecosystem.md#132-subscribe-to-new-blocks-and-to-events)).
2. <a id="soft-2"></a>A batch transaction read, taking a list of transaction
   hashes and returning a result per hash in request order.
3. <a id="soft-3"></a>A block-scoped effects query returning the diffs for every
   transaction in one block, so a consumer following the chain reads one result
   per block rather than one per transaction.
4. <a id="soft-4"></a>Retention policies beyond a single ceiling, such as
   keeping transaction history for accounts a deployment cares about while
   discarding the rest, so a desktop deployment holds what its user needs rather
   than the most recent window of everything.
5. <a id="soft-5"></a>The tips recent transactions chose, per block, so a caller
   chooses a tip against what was paid, should inclusion come to depend on the
   tip. Receipts record the tip each transaction paid (Functionality
   [#68](#f-68)), but for a proof-committed fee that figure includes a surplus
   kept rather than refunded, so this reads the tip each transaction body
   declares.
6. <a id="soft-6"></a>A read of the commitment set that names no commitment: its
   frontier at a given block, with each commitment the viewing key walk
   (Functionality [#89](#f-89)) lists carrying its position, so a wallet keeps
   its own membership witnesses up to date and spends without naming to the node
   the commitments it holds, as a Zcash light client does from the tree state.
   Functionality [#26](#f-26) names them (Functionality [#28](#f-28)).

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
  (Functionality [#83](#f-83), [#84](#f-84)) rather than exposing the protocol.
  That the node takes part in the network is part of the work (see
  [Assumption A3](#a3-following-the-chain-serving-its-history-and-sequencing-are-one-node)).
- **Operating sequencer mode.** The reads and actions specific to running a
  sequencer, such as its committee, approvals, slashing, and cross-zone dead
  letters, are not part of this API. A node in sequencer mode serves the API
  defined here, and how it is operated as a sequencer is outside it.
- **Decoding the actor states of other programs.** The node decodes the native
  balance and the balances of builtin token programs (Functionality
  [#19](#f-19), [#20](#f-20)). Any other program's actor state is returned as
  bytes, and decoding it from the program's IDL belongs to the SDK, as Solana
  leaves the accounts of programs its node does not know to the client.
- **Simulation, and the cost estimate that comes with it.** Executing an
  unsubmitted transaction to see what it would do belongs to the wallet API, for
  both public and privacy-preserving transactions, for the reason given in the
  Design Rationale. Nothing in this RFP executes a transaction a caller
  supplies. The node serves the fee market a caller prices against
  (Functionality [#79](#f-79) and [#80](#f-80)), not a fee for a given
  transaction.
- **The other node functions the surveyed chains offer that LEZ lacks**
  ([Appendix: Blockchain API and SDK Ecosystem, section 4.1](../appendix/blockchain-api-sdk-ecosystem.md#41-absent-on-both-targets)).
  A read-only call executes program code, and is simulation by another name: a
  caller reads the state through the account reads and executes on its own side.
  A pre-submission acceptance check is the wallet's own simulation together with
  the typed rejection of Functionality [#38](#f-38). Waiting for a confirmation
  level is composed from the status query ([#29](#f-29)) and the block
  subscription ([#48](#f-48)) rather than offered as a call that blocks. Batch
  submission is not required: each transaction is submitted on its own.
- **Hosting Logos Core in an application, and reaching a node on another
  machine.** The Logos Core host libraries are a separate component outside this
  RFP, and a node elsewhere is reached through Logos Core peering, which carries
  the module's contract ([#94](#f-94)). This RFP defines the functions, not how
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
  applications (Functionality [#11](#f-11) to [#13](#f-13), [#54](#f-54),
  [#55](#f-55), [#99](#f-99)) and does not change what an event is or which
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
  (Functionality [#37](#f-37)), observes the pending set ([#43](#f-43)), and
  reports its peers ([#83](#f-83)) over the LEZ peer-to-peer network. Today only
  the sequencer joins that network: transaction gossip runs on a topic per zone
  in the sequencer's gossip actor
  (`lez/sequencer/actors/gossip/src/actor.rs:338-340`), and the node joins no
  peer-to-peer network at all. The sequencer service can already run without
  sequencing, as the read-only sequencer the wallet embeds
  (`lez/core_ffi/src/sequencer/service.rs`), following the head and joining
  gossip, which is the closest thing today to a node outside sequencer mode.
- **One node in both modes.** The node and the sequencer run today as separate
  services, each with its own RPC, the node holding the history and the
  sequencer the head. A node in either mode serves the API defined here from
  both. A sequencer's pending set is also only pushed to today
  (`lez/mempool/src/lib.rs:66-92`), so listing it is work in the sequencer. The
  design decision log (Supportability [#11](#sup-11)) records how the two are
  brought together.
- **The head.** The node reads blocks only once they are final on L1
  (`lez/chain_state/src/zone_indexer.rs:54-86`) and has no path for undoing a
  block. Reads at the head ([#96](#f-96)), the head subscription ([#50](#f-50)),
  and the status levels short of final ([#29](#f-29)) need the node to follow
  inscriptions at the L1 tip and to handle a replaced head, as the sequencer
  service does (`lez/chain_state/src/chain.rs:1-3`).

Access control and the contract rest on Logos Core in several places.

- **The contract format.** Modules publish LIDL contracts today. A CDDL
  contract, its description at run time, clients generated from it, and the
  contract hash peering compares are in the draft Logos Core specification, and
  [#94](#f-94) follows whichever has landed.
- **One instance per module.** Logos Core runs one instance of each kind of
  module, while a node follows one zone
  ([Assumption A10](#a10-each-lez-node-follows-one-zone)), so an application
  using several zones restarts the node module to move between them today.
  Supportability [#12](#sup-12) asks for the plan.
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
- **Events and lifecycle per caller.** A Logos Core module event reaches every
  subscriber of the module, and a module is not told when a caller goes away.
  Functionality [#110](#f-110) and Privacy [#4](#priv-4) require delivery per
  caller and [#110](#f-110) requires a disconnected caller's subscriptions to
  end, so the documentation states how a subscription is delivered to the caller
  that opened it and to no other, and how the module learns that a caller has
  gone.

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
