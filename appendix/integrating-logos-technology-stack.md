# Appendix: Integrating the Logos Technology Stack

This appendix describes how an application reaches the Logos technology stack,
and what that costs. It covers the shapes an integration takes, the components
the stack exposes to serve them, the measured size of the node libraries an
embedded integration links, and the questions the stack has not yet settled.

It is written as shared background for the RFPs that define those components
individually, rather than as a specification of any one of them. Where it names
a deliverable, the RFP for that deliverable is the authority on its scope.

## Scope and priority

The Logos technology stack comprises four main components:

- **Logos Blockchain**: the L1 settlement layer.
- **LEZ (Logos Execution Zone)**: programmable money.
- **Storage**: decentralised storage.
- **Delivery**: messaging and delivery infrastructure.

**Regarding integration, this appendix treats LEZ and Logos Blockchain
integration as the primary focus.** An integrator may use either chain
individually, or both together; the analysis and recommendations below are
written with that in mind.

**Storage** is treated as medium priority, with potential integrators already
identified; or as a potential extension to successful LEZ or Blockchain
integration.

**Delivery** is treated as lowest priority at this point in time.

## Integration paths

We attempt to categorise the type of integrations that we want to enable. Such
categorisation helps define technical solutions, as well as prioritising based
on demand and value.

Two choices separate the paths from each other.

**What the application is built from** is the division that matters most. An
application built from Logos modules, a Logos UI module paired with a Logos Core
module, reaches `lez_core` over the Logos Core FFI and needs no development kit
at all. An application not built that way reaches LEZ through a language library
instead. This is a property of the application, not of the platform it runs on:
a Basecamp app is a Basecamp app on desktop, mobile or a server.

**Where the node runs** is the second choice. The application embeds a node,
linking it into its own process, or reaches a remote one over a transport.
Embedding removes a network dependency and a trusted endpoint, at the cost of
both artefact size and resource consumption. The figures under
[Artefact size](#artefact-size) make size the more measurable of the two, but a
running node also spends CPU, network, battery and storage for as long as it is
syncing. Size is paid once at install; the rest is paid continuously, which is
what makes the two costs bite differently on mobile.

This choice is open to both kinds of application. A Basecamp application reaches
the node API over IPC and does not care what answers behind that boundary, so a
local node module and a JSON-RPC client module pointing at a remote node are
interchangeable to it; an application using a development kit selects an
embedded node or a transport client from the same kit.

The choice matters beyond the end-user device. An integrator may run nodes on
behalf of their customers, or deliberately keep node and software separate for
devops reasons even when everything runs within their own infrastructure.

Keeping the choice open matters most on mobile, where none of Logos Blockchain,
LEZ or Storage has settled its mobile strategy yet. Until they do, committing
the architecture to either answer would be premature, so the architecture below
assumes the flexibility is needed and lets the decision be made per platform and
per protocol.

The end goal may resemble the edge and relay distinction that delivery
infrastructure conventionally draws, where a constrained device runs a light
participant that leans on better-provisioned peers, and a capable one carries
the full role. Whether the chain nodes adopt that shape is part of the mobile
topology question under [Open questions](#open-questions).

The three paths below follow from those two choices.

### Basecamp & Logos Core

The Basecamp and Logos Core framework is the canonical way to integrate Logos
technology stack components.

It is proven and working on **desktop platforms** (Linux & Mac, Windows is
actively worked on), as well as **server platforms** with `logosctl`.

**Mobile support** is currently planned for testnet 0.4. Architecture is not
defined at this stage but functionality is expected to be similar as Basecamp
desktop. Meaning a single application that enables the downloading and dynamic
loading of both ui and core modules.

#### Technology Stack

- Business logic can be written in any language as long as FFI bindings can be
  exposed as a "Logos SDK" (C++, Rust, etc). Note that to facilitate developer
  experience, best to have a Logos SDK for the target language. Currently, Logos
  SDK are available in C++, Rust, Nim and JavaScript.
- Canonical UIs are written in QML with C++ Backend, but support for Web
  (QtWebView mobile, QtWebEngine desktop) has been investigated (TODO: clarify
  commitment).

#### Added Value

Logos Basecamp and the core framework are in themselves a valuable product for
secure, private and censorship-resistant distribution of software to users.

- **No reliance on any centralised service**: once Basecamp is installed, the
  whole experience is meant to be over the Logos technology stack: module
  discovery over dedicated DHT, distribution run over Logos Storage (WIP)
- **Secure distribution**, with developer signatures and module integrity checks
  integrated (WIP)
- **Isolated sandboxes** per module, giving a safe execution environment despite
  running native software.
- **Access to the full stack via composition**: messaging, storage and
  blockchain are existing modules to assemble, and an application composed from
  them inherits metadata protection and private state support whether or not its
  developer built anonymity measures directly.
- **Modules are a fault and audit boundary**. Each is developed, audited and
  upgraded independently, and a defect in one does not reach the others.

#### Frictions

- **Distribution via Basecamp/Basecamp as a dependency**: The canonical
  distribution path involves the installation of Basecamp or `logosctl` first.
  Distribution via Basecamp and not App Store for mobile (TODO: to be
  specified). However, there is an available path to **build the application as
  one standalone native app**, linking in all necessary modules. The clearest
  reference is the module tutorial, where `mkLogosQmlModule` wires up
  `apps.default` so that `nix run .` launches a UI module in a standalone window
  with its backend modules bundled, Basecamp not involved, and
  `nix run . --override-input calc_module path:../logos-calc-module` runs it
  against a local module
  ([logos-tutorial, `tutorial-qml-ui-app.md` at `tutorial-v1`](https://github.com/logos-co/logos-tutorial/blob/tutorial-v1/tutorial-qml-ui-app.md)).
  It is currently for development purposes only but it could potentially enable
  distribution both via Basecamp and directly.

- **Language/Framework limitations**: The FFI approach facilitates integration
  in most platforms (browser being the obvious outlier). However, QML is not
  commonly used in the ecosystem. React Native is the most used stack for
  multi-chain wallets (8/10 top wallets; see appendix), with Dart/Flutter and
  native Swift/Kotlin being used by 2 exceptions (among top 10). This does open
  the question of whether **an electron-like experience could convince a
  migration** to Basecamp: specifically, enabling a React development
  experience, and handling the UI and core module wrapping on the developer's
  behalf so they do not write it themselves.

  It is not a straightforward choice. The security goals cut against it, and the
  mobile path is where that bites hardest: QtWebView affords less control than
  the desktop engine, which reintroduces security pitfalls that Basecamp and
  Logos Core are meant to avoid.

#### Current/Potential Gaps

- Completion of Basecamp full USP (security for b)
- Mobile support
- **Electron-like experience/Web UI (?)**: as above, a specific React
  development experience rather than web UI support in general.
- **Standalone app release build (?)**: would an integrator opt for Basecamp and
  still want to distribute their own standalone app? It might make sense on
  mobile, so they can keep their flagship app in the App Store, but it needs
  demand validation: such an integrator gives up some of Basecamp's USP, which
  raises the question of why they would use Basecamp in the first place.

### JSON-RPC Providers + Wallet Library

It is possible to adopt an architecture similar to classic Web3 by exposing APIs
via JSON-RPC. Similar to Eth/Infura, IPFS Gateways, etc.

In this instance, Logos Core CLI can be used to manage Logos nodes (Blockchain,
LEZ, Storage, Delivery).

#### Ecosystem overview

For most blockchains, wallet functionality (signing, proving for LEZ) is
excluded from the JSON-RPC functionalities. `bitcoind` remains the main
exception as of today.

From superficial research, a variety of languages is used by potential
integrators (Rust, Golang, Node.JS, C++, etc.; see
[Appendix: Wallet Application Technology Stacks, section 5](./wallet-tech-stacks.md#5-integrator-infrastructure-unverified)).

From a recent L1 launch perspective, non-EVM chains tend to provide a full suite
of native SDKs (Sui 8 languages, Aptos 4, Injective/Cosmos 4; see appendix). The
SDKs are all native (non-FFI) implementations of their wallet features.

Only Bitcoin uses a Rust-FFI strategy (Bitcoin Dev Kit) to minimize re-writing
of software. Worth noting for interest: BDK generates its Kotlin, Swift, Python
and Dart bindings with [UniFFI](https://mozilla.github.io/uniffi-rs/), Mozilla's
binding generator, from a single Rust codebase. UniFFI solves the binding
surface only and is silent on artefact size, a point the size discussion below
returns to.

The most obvious benefit of a Rust-FFI approach is limiting software rewrite by
maintaining the core logic in one codebase. This has been seen as critical for
Bitcoin due to the limited number of developers (including cryptographers) who
understand Bitcoin protocols and are motivated to implement the libraries. BIPs
are usually implemented in rust-bitcoin first, where Bitcoin maintainers
primarily work. However, there are a number of drawbacks:

1. The cost of wrapping and making the FFI work on various platforms may not be
   as predictable as rewriting the library.
2. The developer experience is less idiomatic, using less type-safety across all
   languages as the API needs to be flat and compatible for all languages;
3. No tree-shaking is available, meaning all functionality is present in the
   pre-built artefact, unless several FFI libraries are provided; especially
   important for mobile platforms.
4. Duplication between libraries: to avoid one big library with no tree-shaking,
   a solution can be to split them (e.g., LDK vs BDK). The resulting issue is
   that both dev kits may ship common code.

The complexity of the wallet software seems to have a strong influence on the
design choice between Sui and Bitcoin. In Sui, the clients mostly sign
transactions, with the chain state being provided by the RPC node. More complex
logic is needed for a Bitcoin wallet: UTXO tracking, building transactions
satisfying complex descriptions (miniscript), and validating them locally.
**Both Logos chains are closer to Bitcoin than to Sui in this regard**, and for
overlapping reasons.

**LEZ** needs to execute private transactions locally in order to prove them.
Execution is itself proven rather than merely performed: each program runs and
is proven in the zkVM, and the composition is then proven recursively.

**Logos Blockchain** transactions carry a zero-knowledge proof in place of a
signature. This holds per operation rather than per transaction: value transfers
and the deposit and SDP operations carry a Groth16 proof, while channel
operations authorise with Ed25519 multisig and a proof-of-work reward claim
carries no proof at all. Value transfer, the common case for a wallet, is in the
proving set.

Zero-knowledge circuits are therefore needed for both chains. They are currently
Circom circuits with C++ witness generators and a C++ prover, exposed over a C
ABI and wrapped in Rust `-sys` crates, shipped from
[`logos-blockchain-circuits`](https://github.com/logos-blockchain/logos-blockchain-circuits).
Those libraries have to be embedded by any wallet that builds a transaction,
whatever language it is otherwise written in.

Private UTXO is not implemented yet. Once it is, it will further complicate the
wallet logic for Logos Blockchain. Beyond proving, a wallet tracks UTXOs, which
brings fork tracking, reorg handling and pending state with it. The external
wallet implementation in the zone SDK is the reference for what that actually
involves.

#### Logos characteristics

There are a few characteristics to note for a Logos wallet library that differs
from other L1s, in addition to being non-EVM:

- 2 chains to integrate: Logos Blockchain the L1, native tokens are native to
  the L1, so is staking; LEZ is the programmable chain living in a zone; anyone
  can create their own zone: Zone-as-a-Service.
- The chains have different cryptographic schemes between LEZ and Logos
  (Secp256k1, ed25519)
- LEZ private transactions require client-side **execution and** proving. LEZ
  public transactions require neither, so the cost is per transaction kind.
- Logos Blockchain value transfers require client-side proving, since the proof
  stands in for the signature.

**If the RPC node and the wallet are run by different entities, are private
transactions still valuable?**

The answer depends on which chain is being used.

**LEZ private transactions.** Mostly yes, with caveats. Submitting a private
transaction to an RPC endpoint does reveal network-level PII (IP address, API
key) and the transaction payload that the RPC receives. That payload includes
commitments and nullifiers, but the actual private account state is not in the
payload: private values are encrypted to the recipient's viewing key and remain
local to the account owner. A recipient with the viewing key can later decrypt
the post-state, but the RPC provider cannot read it without that key. If a
public account is involved in the transaction, the public portion is visible to
the RPC provider as it would be to any node.

**Logos Blockchain transactions.** The value is reduced because the privacy
mechanism is different. The zero-knowledge proof a transaction carries replaces
the signature, not the contents: amounts and output public keys are stored in
the clear, so transaction contents are public on the ledger, similar to Bitcoin.
The Bedrock layer does not hide transaction data; instead, privacy is provided
at the network level by the Blend Network, which hides the identity of block
proposers and blends transactions among core nodes and concurrent transactions
to obscure their origin. Submitting through a third-party RPC bypasses the Blend
Network entirely, because the RPC receives the transaction directly from the
caller before relaying it. The RPC operator therefore sees the full transaction
contents and can correlate them with the caller's IP address and API key. The
only way to benefit from Blend privacy is to submit directly to the peer-to-peer
network.

**Storage and delivery.** Privacy is not primarily provided by the RPC boundary.
It comes from participating directly in the mix, gossipsub, or DHT networks.
Routing reads or writes through a remote RPC provider therefore weakens the
privacy properties those protocols are designed to provide.

Thus, trust assumptions need to be considered when separating the node from the
wallet. When both run inside the same infrastructure (for example, a CEX or
custodian), the trust assumptions are similar to running them on the same
device. However, when the RPC node is run by one entity (a project team or
provider) and the wallet by another (an end-user wallet, DEX, or aggregator),
the RPC provider can see transaction metadata and correlate it with the caller.
That weakens the value of LEZ private transactions and removes the Logos
Blockchain Blend privacy entirely relative to running the node locally.

#### Potential demand

The JSON-RPC provider + wallet library model is likely to be the highest-demand
path for pre-existing applications that cannot adopt Basecamp. Likely consumers
include:

- **Existing mobile wallets** adding LEZ or Logos Blockchain support. They need
  a wallet library in Kotlin, Swift, Dart, or React Native, plus a JSON-RPC
  client, but they **may** not embed a node given mobile size limits and the
  CPU, network and battery a running node consumes. Further clarification of
  blockchain nodes on mobile is needed: see [Open questions](#open-questions).
- **Centralised exchanges, custodians and server-side integrators** that already
  run their own node infrastructure, in Go, Rust, Node.js or Python. They need
  wallet operations (key handling, signing, proving) exposed through a language
  SDK to integrate into their existing backends, and may want separation from
  the nodes to fit their current infrastructure architecture, as well as
  flexibility on the topology (redundancy, load balancing, and so on).
- **RPC providers** that want to expose Logos nodes through a standard transport
  without requiring consumers to use Basecamp. This is a second-order
  integration need: it exists because the integrators above want a third party
  to handle node management rather than running nodes themselves.

### Node and Wallet as a Library

This path is for integrators who want to embed both the node and the wallet in
their own application process, without adopting Basecamp or Logos Core as a
framework. It is the closest to a traditional light-client or embedded-node
model: the application links native libraries that provide chain
synchronisation, state queries, transaction construction, signing, and
submission, all running locally.

It differs from Basecamp in that the integrator does not write business logic as
Logos modules or distribute through Basecamp. It differs from the JSON-RPC
provider path in that there is no remote node: the wallet and node are
co-located in the same process and communicate through in-process APIs rather
than over a transport.

#### Technology Stack

- The node is linked as a native library (for example, `libindexer_ffi.so` for
  LEZ, `liblogos_blockchain_c.so` for Logos Blockchain).
- The wallet is linked as a native library, either from the same crate or a
  separate one.
- Language bindings are generated over the Rust FFI boundary, similar to BDK.
- The application initialises and drives both node and wallet lifecycles
  directly.

#### Added Value

This path carries the Logos value proposition minus what Basecamp and Logos Core
add on top of it.

- **No infrastructure needed**: the integrator runs no nodes and depends on
  nobody else's. There is no endpoint to operate, pay for, or trust.
- **No network dependency for reads or writes**: once synced, the application
  can query chain state and submit transactions without reaching a third-party
  RPC endpoint.
- **Stronger privacy**: the user does not reveal which addresses or transactions
  they care about to a remote RPC provider, and submitting directly to the
  peer-to-peer network is what preserves the network-level privacy the protocols
  provide.
- **Censorship resistance**: the application talks directly to the peer-to-peer
  network.

#### Frictions

- **Artefact size**: embedding a node is expensive. The LEZ indexer FFI library
  alone is ~28 MiB stripped; the Logos Blockchain node library is ~84.5 MiB
  stripped (see [Artefact size](#artefact-size) for provenance). Together they
  exceed typical mobile size budgets before application code or assets are
  added.
- **Build complexity**: integrators must link Rust crates or prebuilt native
  libraries and manage their dependency graphs, including ZK circuit artefacts
  that currently do not build for Android or iOS.
- **Resource consumption**: running a node consumes CPU, memory, battery, and
  storage, which is punishing on mobile.
- **Platform support gaps**: mobile builds of the node libraries are not yet
  available, and the ZK circuits they depend on are not ported to mobile
  architectures.

#### Current/Potential Gaps

- Mobile strategy for LEZ and Logos Blockchain nodes
- Build zk circuits for mobile
- Size optimisation and modularisation of node libraries
- FFI bindings for embedded-node APIs in target languages

### Scenarios

The concrete integrations the three paths produce:

- **A new desktop, mobile or server application on Basecamp.** The recommended
  shape. The application is Logos modules, so it reaches `lez_core` and
  `blockchain_module` through the runtime and inherits whatever those modules
  offer without linking anything itself. Whether those modules run a node
  locally or reach a remote one is a deployment choice it does not encode.
  Mobile is not reachable yet, for the reason under
  [Open questions](#open-questions).
- **An existing mobile wallet adding LEZ.** It cannot become a Basecamp app
  without re-architecting, so it takes the development kit for its language and
  reaches a remote node over JSON-RPC. The wallet runs locally and holds the
  keys; only the node call leaves the device.
- **An existing desktop or server application embedding a node.** The same kit
  with the node selected rather than a client, which suits a deployment that
  would rather not depend on someone else's node and can carry both the size and
  the resources a running node consumes.
- **A centralised exchange or custodian backend.** A server integration, and the
  reference profile the node API requirements are written against. It holds
  accounts, watches for deposits, and confirms transactions before crediting
  them; it holds keys but is not a wallet application, and it is the strictest
  reader of the set.
- **An RPC provider.** Runs nodes and exposes them to others through a transport
  proxy, so it consumes the node API in order to serve it onward rather than to
  act on the chain itself.
- **An integration spanning several zones.** A bridge, an aggregator or an
  explorer covering more than one LEZ zone. How the stack serves this shape is
  an open question: see [Multizone integration](#multizone-integration).

A hybrid is also possible, with the LEZ node embedded and the Logos Blockchain
node remote. At this stage there is no clear justification for such an approach,
but that may change as the mobile strategy for both nodes is clarified.

## Existing Logos SDKs

Two different libraries are involved, and they answer different questions.

**`logos-liblogos` is the host runtime.** It provides `liblogos_core`, a C-API
shared library, and `logos_host`, the module subprocess host binary. It
discovers modules, resolves dependencies, loads them and manages their
lifecycle, and it is consumed by `logos-basecamp` and `logos-logoscore-cli`.

**The language SDKs are guest-side.** They are not wallet libraries in the BDK
sense and not module loaders either: they are what a module uses to implement
its own contract and call the modules it depends on. The four that exist,
`logos-rust-sdk`, `logos-cpp-sdk`, `logos-nim-sdk` and `logos-js-sdk`, all bind
the `lp_*` C ABI exported by `logos-protocol`.

So "a Logos SDK for language X" is shorthand hiding more than one piece of work.
By default the loader spawns a separate OS process per module; for mobile and
embedded use the runtime supports **Local mode**, registering modules in-process
through a `PluginRegistry`.

### Standalone applications hosting modules

An application that is not a Basecamp app and is not itself a module can load
Logos modules and use them, but not through the language SDKs, and the two
halves are served by different targets. **Hosting** is `liblogos_core`
([`logos-liblogos`](https://github.com/logos-co/logos-liblogos), the host
runtime under `src/logos_core/`), whose C API carries the whole lifecycle
including the access policy that gates which caller may invoke which target; the
C++ SDK ([`logos-cpp-sdk`](https://github.com/logos-co/logos-cpp-sdk)) names it
`logos-cpp-sdk::logos_host` and lists a standalone app among its consumers, so
this is an anticipated shape rather than an accident of the API being public.
**Calling** is a separate target, `logos-cpp-sdk::logos_consumer`, since
`logos_core.h` exposes no invoke function at all. The `lp_*` C ABI both bind is
exported by [`logos-protocol`](https://github.com/logos-co/logos-protocol).

**This path is C and C++ today.** No binding for another language reaches
either. The wrappers that exist elsewhere drive the CLI as a subprocess rather
than linking the library, which is usable for a daemon but carries costs an
embedded integration would not accept. The language SDKs do not close the gap:
their `lp_*` symbols resolve inside a module built on the cdylib path, and a
plain binary has no such link step. So the gap for a standalone host in Rust,
Go, Kotlin, Swift or Dart is a binding over the `liblogos_core` and consumer C
APIs, which is new work over a C surface that already carries the semantics a
host needs.

**Two proofs of concept explore this**, and both are exploratory rather than
supported.
[`liblogos-rust-poc`](https://github.com/fryorcraken/liblogos-rust-poc) embeds
`liblogos_core` in a standalone Rust application, driving the full lifecycle and
bringing up a real module with its dependency; it carries a twenty-line C++ shim
constructing the `QCoreApplication` the C API requires but cannot create.
[`liblogos-electron-poc`](https://github.com/fryorcraken/liblogos-electron-poc)
goes further, packaging the runtime and its modules into a single distributable
artefact and measuring each step by reproducible experiment. Its
[0.3.0 inventory](https://github.com/fryorcraken/liblogos-electron-poc/blob/main/docs/0.3.0-inventory.md)
is the most detailed account available of what hosting the runtime in process
costs, and the sections below draw on it. It remains Linux only.

#### What a development kit must provide

Three capabilities, currently spread across three repositories, are what a
library must offer in a given language. An integrator needs all of them before
anything works end to end, since the runtime lifecycle alone loads modules it
cannot call.

**1. Runtime lifecycle.** Discovery, dependency resolution, loading, access
policy. The `logos_core_*` C ABI from `logos-liblogos`. Solved: both proofs of
concept wrapped it without difficulty.

**2. Provider hosting.** Registering a provider so calls can be served, through
`LogosAPI` and `LogosAPIProvider`. This is the barrier: `LogosAPI` is a
`QObject` with no C ABI over it, so every binding needs a C++ shim and a Qt one.

**3. Invocation and authorisation.** Calling methods, subscribing to events, and
the capability handshake. Half solved: the provider interface already defines a
universal string and JSON interface alongside the Qt one, but the hosting side
and the token handshake are still Qt C++.

An in-process caller appears to need neither the gateway nor a token, which
would make the third capability smaller than it looks. The Electron proof of
concept's
[0.3.0 inventory](https://github.com/fryorcraken/liblogos-electron-poc/blob/main/docs/0.3.0-inventory.md)
measured the irreducible C++ at roughly **120 lines**, with the gateway,
provider registration, token manager and TCP transports all excluded. That
figure comes from one proof of concept on one platform, so it is a starting
estimate rather than a scope (TODO: confirm with @dlipicar).

The Qt dependency bears on the three differently: a build-time cost for the
first, the barrier itself for the second, and what keeps an already universal
interface out of reach for the third. A C ABI over the universal interface would
address the Qt dependency, the C++ ABI commitment and the event loop question
together, though whether one change covers all three needs confirming. Qt
removal is an active programme rather than a proposal, including removal of the
`QCoreApplication` requirement
([logos-logoscore-cli#35](https://github.com/logos-co/logos-logoscore-cli/pull/35)),
which is what the Rust proof of concept had to shim around. Roadmap pages
predate much of this work, so an absent item there is not evidence a piece is
unplanned.

Beyond the binding itself, shipping a library rather than a demonstration
carries platform and packaging work that is easily underestimated: the event
loop integration demonstrated on Linux does not carry to Windows or macOS
unchanged, a library is called in orders an application never attempts, prebuilt
binaries are needed per platform and per runtime version, and plugin libraries
opened by path at run time are invisible to ordinary dependency inspection.

### Fit for the JSON-RPC Provider + Wallet Library model

The JSON-RPC proxy is a Logos Core module deployed via `logosctl` alongside the
node modules it fronts. That part is standard Logos module development and does
not need separate SDK analysis.

The more relevant question is whether a **wallet module built with a Logos SDK**
can consume a remote JSON-RPC API:

- **Logos Blockchain.** The wallet service runs node-side and exposes an HTTP
  API. `wallet-http-client` (~115 LOC) is a consumer-side HTTP client for that
  API, not an internal communication channel used by the `wallet` crate itself.
  The API is HTTP, not JSON-RPC, so pointing a wallet module at a remote
  JSON-RPC endpoint would require structural changes or a bridge.
- **LEZ.** The `lez/wallet` crate reaches the node through internal Rust APIs
  (`lez/sequencer/service/rpc`), not over HTTP. Remote JSON-RPC consumption is
  unverified.

**Strengths:**

- A wallet built as a Logos Core module inherits sandboxing and lifecycle
  management.
- The same wallet module can run in Basecamp or in a standalone module host.

**Gaps:**

- LEZ wallet-to-node communication is internal; remote JSON-RPC consumption is
  unverified.
- Logos SDKs do not provide a generic JSON-RPC client for non-module
  applications.
- No Kotlin, Swift, Dart, or Go SDK exists for wallet modules.

### Fit for the Node and Wallet as a Library model

The existing modules already carry node and wallet functionality and can be
reused across Basecamp and embedded integrations, but hosting them is reachable
from C and C++ only, mobile Local mode is undelivered, and artefact size and
tree-shaking are unresolved for mobile. What follows from that is in
[Recommendation](#recommendation).

## Recommendation

Given the analysis above, one path is recommended and a second is held in
reserve. Relying exclusively on the Logos SDK is the end state worth building
towards; a per-module FFI-wrapped kit is the fallback if that proves
unmanageable. A third approach, native wallet libraries per language, is
discarded outright, and the reasoning is recorded because its absence would
otherwise look like an oversight.

**Basecamp & Logos Core** remains the canonical path for new applications. It is
not discussed below because it needs no separate native library or development
kit; applications are built from Logos modules directly.

### Ideal long-term: rely exclusively on the Logos SDK

Make the Logos module stack the only integration surface and avoid creating
dedicated wallet or node native libraries altogether. An application would host
modules through `liblogos_core` and call them through the consumer surface,
which is the arrangement Basecamp and `logosctl` already use.

This means a library per language carrying the three capabilities set out under
[What a development kit must provide](#what-a-development-kit-must-provide):
consuming a module's API, managing modules, and the token and capability
handshake. A delivery covering fewer than all three leaves an integrator with
something that does not work end to end, so they belong in the same piece of
work.

This requires:

1. **Make the host runtime reachable from the target languages.** Hosting is
   reachable from C and C++ only today, so this path needs bindings over
   `liblogos_core` and the consumer surface for Kotlin, Swift, Dart, Go and
   Rust. This is more than wrapping a C header: the Qt dependency leaks through
   the C boundary, so each binding currently needs a C++ shim before it can call
   the API at all. See
   [What a development kit must provide](#what-a-development-kit-must-provide)
   for the capabilities this breaks into.

2. **Resolve Android and iOS risks.** Deliver mobile Local mode and confirm that
   modules can run in-process on Android and iOS.

3. **Separate wallet and node functionality into distinct modules.** Split
   `lez_core` and `logos-blockchain` into smaller modules so that a wallet
   module can be loaded without the full node. This enables smaller node
   artefacts for mobile apps (wallet-only integration) and makes wallet + node
   combinations possible on demand. On Android, each module should be shipped as
   an independent AAR, similar to the LEZ wallet AAR described below, so that
   standalone mobile applications can include only the modules they need and
   leave out the rest.

4. **Enable dynamic configuration between remote JSON-RPC nodes and local
   nodes.** With modular wallet and node components, an application could load a
   wallet module locally, point it at a remote node module via JSON-RPC, or load
   a local node module, switching at runtime based on platform, network
   conditions, or user preference. This enables both modes for non-Basecamp apps
   at low cost.

This is the ideal end-state because it avoids maintaining parallel SDKs (Logos
SDK + LEZ-DK + Logos Blockchain SDK) and lets integrators compose only the
components they need. It also relies on the Logos core framework, having most
components in common with the Basecamp path (TODO: verify/clarify). However, it
depends on delivering mobile Local mode, modularising the node crates, and
producing language SDKs for Kotlin, Swift, Dart, and Go with good tree-shaking
capabilities. If those prerequisites are not met, this path is not yet viable.

### Discarded: native wallet libraries per language

Before the recommendation below, one approach is worth ruling out explicitly,
because it is the common one elsewhere and its absence here would otherwise look
like an oversight. Recent non-EVM L1s ship native wallet libraries in each
language they support: Sui in eight, Aptos in four, Injective and Cosmos in
four, and Ethereum and Solana likewise have mature per-language wallet stacks
that re-implement signing rather than binding one codebase. Only Bitcoin takes
the Rust FFI route, through the Bitcoin Development Kit.

**Neither Logos chain fits the first pattern, so this appendix discards it for
both.** The reasoning that makes Bitcoin the exception applies here as well, and
in some respects more strongly.

Bitcoin's case is that its wallet logic requires deep protocol and cryptographic
knowledge held by a small number of people, that BIPs land in `rust-bitcoin`
first, and that re-implementing them per language multiplies both the work and
the opportunity for a subtle, consensus-relevant mistake. A native library in
another language is not merely more code; it is more code of a kind that is hard
to get right and hard to review.

Both Logos chains carry that property:

- **LEZ** requires client-side proving for private transactions, so a wallet
  executes and proves locally rather than signing a payload.
- **Logos Blockchain** replaces signatures with zero-knowledge proofs whose
  circuits ship as C libraries. Any wallet that builds a transaction embeds
  those libraries whatever language surrounds them, so "native" would mean a
  language-specific shell over the same C dependency rather than an independent
  implementation. Private UTXO increases that footprint. Alongside proving, UTXO
  tracking brings fork handling, reorg and pending state, which is the part of
  Bitcoin wallet software that is most often reimplemented and most often
  reimplemented wrongly.

The consequence is that a pure native wallet library in Kotlin, Swift, Dart or
Go is discarded for both chains. It would duplicate cryptography that is
difficult to write correctly, while still embedding the C circuit libraries it
was meant to avoid, so it would carry the cost of a rewrite without the benefit
of independence.

A specification is the one thing that would change this calculus, and it is
worth keeping open rather than pursuing now. A written specification for UTXO
tracking and transaction construction, extracted from the existing Rust
implementation, would let a native library be built against something reviewable
rather than against a reference implementation's behaviour. Whether such a
specification can be usefully extracted is unresolved.

### Fallback: a per-module FFI-wrapped development kit

If the Logos SDK path proves unmanageable, whether because the Qt dependency
cannot be lifted or because some part of the generic runtime turns out not to be
reachable from the target languages, there is a simpler shape that reaches the
same integrators.

Rather than binding the generic runtime, wrap every module individually over
FFI, and write the glue that composes them in the native language rather than
loading modules dynamically. The result is a static, non-generic liblogos: it
does not discover modules or expose their interfaces at run time, it just wires
a known set of them together. The wrapping is done per component precisely so
that the native toolchain can drop the wrappers, and ideally the `.so` files
behind them, for components an integrator does not use.

Two things recommend it as a fallback. It is incremental in a way the SDK path
is not: the LEZ wallet can be wrapped first and the Logos Blockchain wallet
after it, with each delivering value before the next starts. And it avoids the
generic loading machinery altogether, so the problems that would have blocked
the SDK path do not arise.

The cost is that it is still a lot of development to produce something less
powerful than liblogos, and it is paid per language rather than once. That is
why it sits here rather than above: it is what to do if the SDK path is
foreclosed, not a cheaper route to the same place.

This approach still profits from the wallet and node separation required of both
blockchain libraries. That work is a prerequisite for either recommendation, not
a property of one, so it is not lost if the SDK path is abandoned: splitting
`lez_core` and `logos-blockchain` so a wallet can be built without the full node
is what lets a per-module kit wrap a wallet on its own and keep the node out of
a mobile artefact.

Tree-shaking is the part that has to work for this to pay off, and Android is
where it is hardest: R8 and ProGuard remove unused bytecode but do **not**
remove a dynamic `.so` from the APK, and every exported FFI symbol is a
dynamic-linker entry point the linker cannot prove dead. iOS is more forgiving,
since a static `.a` can be dead-stripped at app-link time. BDK is the worked
example of the Android case: `bdk-android` 3.1.0 ships one monolithic
`libbdkffi.so` per ABI, 42.6 MiB of native code across three ABIs against 1.7
MiB of Kotlin, with minification disabled. UniFFI generates the bindings but
offers nothing for size; where comparable projects control it, they do so with
build-time feature gating, as LDK Node does with Cargo features.

Per-module artefacts are therefore the mechanism, separate AARs per wallet on
Android and separate Frameworks or SPM products on iOS, rather than one artefact
the toolchain is expected to trim.

The figures under [Artefact size](#artefact-size) are what this has to beat.

## Deliverables

Five components make LEZ integrable by the parties that have to integrate a
chain before it is usable in practice: wallets, exchanges, custodians, payment
gateways, aggregators, RPC providers, ramps, bridges and accounting providers.
Each acts on the chain from outside it and stops at the first capability that is
missing. The first two define surfaces; the rest consume them. Each has its own
RFP, and each of those is the authority on its own scope.

1. **The LEZ node API** ([RFP-027](../RFPs/RFP-027-lez-node-api.md),
   [logos-co/ecosystem#235](https://github.com/logos-co/ecosystem/issues/235)).
   The global, non-wallet functions, packaged in the `lez_core` Logos Core
   module. Exposed in Rust for the wallet features within `lez_core`, and over
   the Logos Core FFI for Basecamp apps and transport proxy modules (see 3 and
   5).

2. **The LEZ wallet API**
   ([logos-co/ecosystem#236](https://github.com/logos-co/ecosystem/issues/236)):
   key handling, derivation, proving, and signing, covering local and wallet
   operations only. Packaged in the `lez_core` module, which can run with every
   wallet API function disabled. Exposed over the Logos Core FFI and through the
   `lez_wallet_ffi` crate so it can be wrapped for other languages (see 4).

3. **The JSON-RPC proxy module and its client library**
   ([logos-co/ecosystem#237](https://github.com/logos-co/ecosystem/issues/237)):
   A Logos Core module projecting the `lez_core` APIs over JSON-RPC, carrying
   both the node and wallet APIs, plus a Rust client library for that surface.

   The client should also be packaged as a **Logos Core module** in its own
   right, exposing the LEZ node API over the Logos Core FFI and answering it by
   calling a remote endpoint. That keeps a Basecamp application on one shape:
   its core module always reaches the node API over IPC, and what sits behind
   that boundary is either a local node module or the client module pointing at
   a remote node. Local and remote become a deployment choice rather than a code
   change, which is what lets a mobile Basecamp application skip the node
   artefact entirely while still speaking the same API.

4. **LEZ-DK: The LEZ Development Kit**
   ([logos-co/ecosystem#238](https://github.com/logos-co/ecosystem/issues/238)):
   Modelled on the Bitcoin Development Kit (BDK), for the reasons set out in
   [RFP-027: Inspiration from BDK](../RFPs/RFP-027-lez-node-api.md#inspiration-from-bdk-bitcoin-development-kit).
   It carries the FFI crates exposing the Rust node and wallet APIs, covering
   both an in-process node and the client for a remote one, and ships per
   language, Kotlin, Swift and Go among them.

   Whether the in-process node belongs in the same kit or a separate one is yet
   to be decided; it drives the artefact size, so the choice turns on the
   figures under [Artefact size](#artefact-size).

   The LEZ-DK exists to integrate LEZ into applications that already exist. It
   is not the recommended starting point for something new, where the Logos Core
   framework and Basecamp are the encouraged path.

5. **Further transport proxy modules and their client libraries**
   ([logos-co/ecosystem#222](https://github.com/logos-co/ecosystem/issues/222))
   beyond JSON-RPC, such as gRPC, GraphQL, and a Mesh or Rosetta adapter. Each
   adds a Logos Core module exposing the wallet and node APIs over the new
   transport, and a Rust client library with its FFI crate, consumed through the
   LEZ-DK.

   The shape generalises: a server module projecting a node's API over a
   transport, paired with a client module re-exposing the same API over the
   Logos Core FFI, applies to any transport and to any node's API. A consuming
   module speaks IPC in every case and does not learn which transport carried
   the call, so transports can be added without the applications above them
   changing.

   **Delivery is the pair worth singling out**, because it removes a
   prerequisite the others keep. Every other transport needs the node reachable
   at an address: a public endpoint, a forwarded port, a configured router, or a
   third party running the node. Delivery is the Logos messaging layer, so a
   client and server pair over it reaches a node without any of that. That bears
   on the mobile topology question: a user running a node at home could recover
   on a phone whatever a mobile build gives up, so the sacrifices a mobile build
   makes would bind only those without a node of their own rather than everyone.

Note that similar components are needed for Logos Blockchain integration and
will be defined in future RFPs, tracked as the FFI bindings
([logos-co/ecosystem#219](https://github.com/logos-co/ecosystem/issues/219)) and
the proxy module and its SDK
([logos-co/ecosystem#220](https://github.com/logos-co/ecosystem/issues/220)),
both under
[logos-co/ecosystem#214](https://github.com/logos-co/ecosystem/issues/214). The
intent is for the JSON-RPC proxy module to cover the Logos Blockchain API as
well, and other module APIs alongside it.

Two consequences bear on the node API. It is consumed both directly, by a caller
holding it across an FFI boundary, and indirectly, through the JSON-RPC proxy
and the client library, so its surface has to survive projection onto a wire
protocol rather than assuming a local caller. And because a consumer may reach
the node through either path, the two must express the same semantics.

### The LEZ node API is the only API for LEZ chain state access

The node API is the whole of the surface available to a consumer, in both
directions, and a LEZ node is a black box: which part of it answers a given call
is implementation, not API. The node serves every read, obtaining what it does
not hold rather than directing the consumer elsewhere. It is also the only way
onto the chain, relaying signed transactions and new commitments produced by the
wallet without constructing or signing anything itself. A capability a consumer
needs is therefore required of the node regardless of which part of it holds the
data today.

### Wallet and node separation

Wallet and node functionality must be separable for both chains, so that a
wallet can be loaded without the full node. This is what makes a wallet-only
mobile integration possible, and it is a prerequisite for both recommendations
above rather than a property of either one.

## Multizone integration

Anyone can create a zone: Zone-as-a-Service is a property of the LEZ design, so
an integrator covering the ecosystem rather than a single application is likely
to face more than one zone. Bridges, aggregators, explorers, exchanges listing
assets from several zones, and wallets showing a user their holdings across
zones all share this shape.

Whether to facilitate many LEE zones is a commercial question, not only a
technical one, and it should be settled before the architecture hardens. More
zones fragment liquidity, a cost borne by the protocols that most need depth,
and thin markets are a security consideration as much as a user experience one.
Against that, zones give isolation, independent throughput and room for
configurations a shared chain would not accept. A zone is also not obliged to
run LEZ, and one that does not is a different chain for integration purposes,
which bounds what multizone support can promise.

`lez_core` serves one zone at a time, so multizone is an unresolved question for
the architecture rather than a configuration detail. Either the module holds
several zones and the node API gains a zone parameter, or the integrator runs
one instance per zone and the proxy routes between them. Both may end up
requiring a zone identifier in the node API, so the real distinction is where
the multiplexing happens. The indexer's Rust FFI is handle-based, which suggests
the single-instance restriction sits in the C++ module wrapper rather than in
the FFI beneath it; this needs confirming against the source. Until it is
settled, an integrator covering several zones runs a node per zone and joins the
results itself.

Key management should be zone-abstract, and this is a requirement rather than an
option: a user should hold one seed for Logos, not one per zone and not one per
chain. The same applies across LEZ and Logos Blockchain, where it is the more
common case, but the cryptography constrains it. The two chains use different
schemes, secp256k1 and ed25519, and BIP-32 public derivation works on the former
while SLIP-0010 ed25519 permits only hardened derivation
([Appendix: Wallet Libraries Ecosystem, section 3](./wallet-libraries-ecosystem.md#3-key-derivation)).
A single seed can feed both, but watch-only derivation is available on one side
and not the other, and LEZ viewing keys are further key material a user expects
to recover from their seed. What a clean API looks like over this, where the
derivation tree branches, and whether hardware-backed keys and external signers
are in scope are open questions for R&D. Hardware signing bears on LEZ more than
on the L1, since client-side proving is not obviously something a signing device
performs.

## Artefact size

Artefact size is the constraint that bears hardest on any kit shipping a node,
and the figures below are what the recommendations above are measured against.

The LEZ node alone already sets a high floor. Built from `lez/indexer/ffi` at
`--release` for `x86_64-unknown-linux-gnu`, `libindexer_ffi.so` is 33.6 MiB
unstripped and 28.0 MiB stripped. For comparison, `bdk-android` 3.0.0 ships a
15.3 MiB `.so` for `arm64-v8a`, and that one artefact carries a wallet and four
chain backends. So the LEZ node on its own, before any wallet or client joins
it, already outweighs a complete Bitcoin kit. This is a verifier only: the risc0
`prove` feature is enabled by `lez/wallet-ffi` and a benchmark tool alone, and
`indexer_ffi` resolves `risc0-zkvm` with `client` and `std`, so the figure
excludes proving. What it does include is the risc0 verifier, RocksDB, libp2p,
and the guest ELF the state machine embeds.

A LEZ node also needs a Logos Blockchain node, and that one is larger again.
`logos-blockchain-c` builds `liblogos_blockchain.so` at 84.5 MiB, also for
`x86_64-unknown-linux-gnu`, already stripped by its release profile, since it
carries the whole node: the in-house `zk/` circuit subtree with its provers,
plus the chain, blend, proof-of-work, wallet and API services. An application
embedding both a LEZ node and the L1 node it reads from therefore starts from
roughly 112 MiB for a single protocol pair, which is the same order as the size
ceilings the mobile stores impose, before the application has shipped any code
of its own. That is also before Storage and Delivery, and before any
per-architecture packaging.

Neither number above is for a mobile target, and neither can be today: the ZK
circuit artefacts both libraries depend on are published for Linux, macOS and
Windows only, so an Android or iOS build fails before it produces a size.
Building those circuits for mobile is a prerequisite to answering the question,
and a proposal is expected to obtain the figures rather than assume them.

## Open questions

**Basecamp mobile readiness.** Mobile support is planned for testnet 0.4, so the
Basecamp shape is reachable on desktop and server only at the time of writing.

**Mobile strategy for LEZ and Logos Blockchain nodes.** It is not settled
whether a mobile device runs light versions of the chain nodes and joins the
peer-to-peer networks directly, reaches remote nodes over RPC alone, or combines
the two. None of the protocols has defined its mobile strategy, and the answer
need not be the same for each. The architecture above therefore keeps both
options open: the node API is reached the same way whether a local node module
or a client module answers it, so settling on either answer does not invalidate
the surface built against it.

**Building the ZK circuits for mobile**, which is a prerequisite to obtaining
any mobile artefact size at all.
