# Appendix: Integrating the Logos Technology Stack

This appendix describes how an application reaches the Logos technology stack,
and what that costs. It covers the components the stack exposes, the shapes an
integration takes depending on what the application is built from, the measured
size of the node libraries an embedded integration links, and the questions the
stack has not yet settled.

It is written as shared background for the RFPs that define those components
individually, rather than as a specification of any one of them. Where it names
a deliverable, the RFP for that deliverable is the authority on its scope.

## Target architecture and the LEZ-DK

The five deliverables exist to make LEZ integrable by the parties that have to
integrate a chain before it is usable in practice: wallets, centralised
exchanges, custodians, payment gateways, data and price aggregators, node and
RPC providers, fiat on and off ramps, bridges, and tax and accounting providers.
Each acts on the chain from outside it, holding accounts, watching for what
arrives, signing what it sends, and reconciling against its own books. Each
stops at the first capability that is missing.

A CEX backend and a self-custodial mobile wallet have different architectures
and are written in different languages, which is why the suite takes a
composable approach. The first two deliverables define surfaces; the rest
consume them.

1. **The LEZ node API** ([RFP-027](../RFPs/RFP-027-lez-node-api.md),
   [logos-co/ecosystem#235](https://github.com/logos-co/ecosystem/issues/235)).
   Define the LEZ node API: the global, non-wallet functions. An integrator may
   run a LEZ node and access it through this API by way of a transport proxy,
   which is what an RPC provider does. Whether indexer or sequencer features
   answer a given call is internal to the node, which is a black box in that
   regard. This is the only component used to read blockchain state and to push
   signed transactions and new commitments to the chain. It is packaged in the
   `lez_core` Logos Core module. The API needs to be exposed in Rust, to be
   consumed by the wallet features within `lez_core`, and over the Logos Core
   FFI, to be consumed by Basecamp apps and transport proxy modules (see 3 and
   5).

2. **The LEZ wallet API**
   ([logos-co/ecosystem#236](https://github.com/logos-co/ecosystem/issues/236)):
   key handling, derivation, proving, and signing. It covers the local and
   wallet operations only. The LEZ wallet library that exposes this API is
   packaged in the `lez_core` module, and an integrator can run that module with
   every wallet API function disabled. The API needs to be exposed over the
   Logos Core FFI on the `lez_core` module, and through the `lez_wallet_ffi`
   crate so it can be wrapped in libraries for other languages (see 4).

3. **The JSON-RPC proxy module and its client library**
   ([logos-co/ecosystem#237](https://github.com/logos-co/ecosystem/issues/237)):
   The module is a Logos Core module that projects the APIs the `lez_core`
   module exposes over JSON-RPC, carrying both the LEZ node API and the LEZ
   wallet API, the former being the one an integrator running a LEZ node as an
   RPC provider uses most. The client is a Rust library for that same surface,
   so a consumer reaches a remote node without writing the transport itself.

4. **LEZ-DK: The LEZ Development Kit**
   ([logos-co/ecosystem#238](https://github.com/logos-co/ecosystem/issues/238)):
   Modelled on the Bitcoin Development Kit (BDK), for the reasons set out in
   [RFP-027: Inspiration from BDK](../RFPs/RFP-027-lez-node-api.md#inspiration-from-bdk-bitcoin-development-kit).
   The LEZ-DK carries the FFI crates that expose the Rust LEZ node API and LEZ
   wallet API. It covers both direct access to an in-process LEZ node (see the
   embedded-node example below) and the JSON-RPC client library for a remote one
   (see the Android example below). It ships as a unified library per language,
   Kotlin, Swift and Go among them, so an application integrates LEZ the same
   way whether the node runs in process or remotely.

   Whether the in-process node belongs in the same kit as the wallet and the
   clients, or in a separate one, is yet to be decided. It is the component that
   drives the artefact size, so the choice turns on the figures discussed in
   [Unified Logos Development Kit & artefact size](#unified-logos-development-kit--artefact-size).

   The LEZ-DK exists to integrate LEZ into applications that already exist, and
   to help LEZ reach the users those applications already have. It is not the
   recommended starting point for something new. A developer building a new
   application, whether or not LEZ is the whole of it, is strongly encouraged to
   build on the Logos Core framework and Basecamp instead, which is the first of
   the shapes below rather than the second.

5. **Further transport proxy modules and their client libraries**
   ([logos-co/ecosystem#222](https://github.com/logos-co/ecosystem/issues/222))
   beyond JSON-RPC, such as gRPC, GraphQL, and a Mesh or Rosetta adapter. Each
   adds a Logos Core module exposing the wallet and node APIs over the new
   transport, and a Rust client library with its FFI crate, consumed through the
   LEZ-DK.

Note that similar components are needed for Logos Blockchain integration and
will be defined in future RFPs, tracked as the FFI bindings
([logos-co/ecosystem#219](https://github.com/logos-co/ecosystem/issues/219)) and
the proxy module and its SDK
([logos-co/ecosystem#220](https://github.com/logos-co/ecosystem/issues/220)),
both under
[logos-co/ecosystem#214](https://github.com/logos-co/ecosystem/issues/214). The
intent is for the JSON-RPC proxy module to cover the Logos Blockchain API as
well, and other module APIs alongside it.

The three diagrams below show the architecture these deliverables build towards.
They differ in what the application is built on, which is the division that
matters, and then in where the node runs. A new app built on Basecamp is a Logos
UI module paired with a Logos Core module, and its core module reaches the
`lez_core` module over the Logos Core FFI, on any platform it runs on. A
pre-existing application not built from Logos modules uses the LEZ-DK for its
language instead, and from there either reaches a remote node over a transport
or embeds one of its own.

The `lez_core` module exposes both surfaces through one Logos Core FFI, and the
app's core module consumes both, the wallet for keys and signing and the node
for chain state:

```mermaid
flowchart TB
  subgraph app["New app (wallet, DEX, etc): uses Basecamp and Logos Core"]
    direction TB
    appUi["UI module"] --> appCore["Core module"]
  end

  subgraph lezmod["lez_core module"]
    direction TB
    ffi["lez_core Logos Core FFI"]
    walletApi["LEZ wallet API"]
    nodeApi["LEZ node API"]
    wallet["LEZ wallet (Rust)<br/>includes sync"]
    node["LEZ node (Rust)"]

    ffi --> walletApi
    ffi --> nodeApi
    walletApi --> wallet
    nodeApi --> node
    wallet -- "LEZ node Rust API" --> node
  end

  subgraph lbmod["blockchain_module"]
    direction TB
    lbFfi["blockchain_module Logos Core FFI"]
    lbApi["Blockchain node API"]
    lbNode["Blockchain node (Rust)"]

    lbFfi --> lbApi
    lbApi --> lbNode
  end

  appCore --> ffi
  node --> lbFfi

  style ffi fill:#ffffff,stroke:#999999,stroke-dasharray:3 3
  style walletApi fill:#ffffff,stroke:#999999,stroke-dasharray:3 3
  style nodeApi fill:#ffffff,stroke:#999999,stroke-dasharray:3 3
  style lbFfi fill:#ffffff,stroke:#999999,stroke-dasharray:3 3
  style lbApi fill:#ffffff,stroke:#999999,stroke-dasharray:3 3
```

An existing Android application, a wallet already shipping that is adding LEZ
support, is not built from Logos modules. It adds one dependency, the LEZ-DK for
Kotlin, which carries the wallet and the transport clients, each its own FFI
crate over its own Rust crate. The application holds the wallet and a client and
wires them together, choosing the transport it reaches the node through, and
neither component reaches the other. Every client in the kit is linked whether
or not it is used, which is the cost of shipping one artefact. The client runs
inside the application rather than beside it, so only the node call leaves the
device:

```mermaid
flowchart TB
  subgraph android["Integration in a pre-existing Android wallet"]
    direction TB
    androidApp["Application (Kotlin)"]

    subgraph devkit["LEZ-DK for Kotlin"]
      direction TB
      walletFfi["lez_wallet_ffi"] --> walletLib["LEZ wallet"]
      jsonFfi["lez_json_client_ffi"] --> jsonClient["json_rpc_lez_client"]
      grpcFfi["lez_grpc_client_ffi"] --> grpcClient["grpc_lez_client"]
    end

    androidApp -- "LEZ wallet Kotlin API" --> walletFfi
    androidApp -- "LEZ node Kotlin API" --> jsonFfi
  end

  subgraph remote["Remote host (logosctl)"]
    direction TB
    proxy["JSON-RPC proxy module"] -- "LEZ node API (Logos Core FFI)" --> rnode["LEZ node module"]
    rnode -- "Blockchain node API (Logos Core FFI)" --> rlbmod["blockchain_module"]
  end

  jsonClient -- "LEZ node API (JSON-RPC)" --> proxy

  style grpcFfi fill:#eeeeee,stroke:#bbbbbb,color:#999999
  style grpcClient fill:#eeeeee,stroke:#bbbbbb,color:#999999
```

An application can also link the node itself rather than reach one over a
transport, which is the same LEZ-DK with a different component selected. A Dart
and Flutter wallet in the shape of
[Cake Wallet](https://github.com/cake-tech/cake_wallet) adds the node FFI beside
the wallet FFI and runs both in process, so the client stays linked but unused
and no node call leaves the device:

```mermaid
flowchart TB
  subgraph embedded["Integration in a pre-existing wallet app"]
    direction TB
    embeddedApp["Application (Dart / Flutter)"]

    subgraph devkit["LEZ-DK for Dart"]
      direction TB
      walletFfi["lez_wallet_ffi"] --> walletLib["LEZ wallet"]
      nodeFfi["lez_node_ffi"] --> nodeLib["LEZ node"]
      jsonFfi["lez_json_client_ffi"] --> jsonClient["json_rpc_lez_client"]
    end

    embeddedApp -- "LEZ wallet Dart API" --> walletFfi
    embeddedApp -- "LEZ node Dart API" --> nodeFfi
  end

  style jsonFfi fill:#eeeeee,stroke:#bbbbbb,color:#999999
  style jsonClient fill:#eeeeee,stroke:#bbbbbb,color:#999999
```

**Note**: unlike the two above, this diagram omits the Logos Blockchain node
that the embedded LEZ node reads finalised on-chain state from. Whether an
application embedding a LEZ node should embed the L1 node as well, or reach a
remote one, is undecided: see
[Unified Logos Development Kit & artefact size](#unified-logos-development-kit--artefact-size)
for the sizes that bear on the choice.

Each component has its own RFP, and each of those is the authority on its own
scope.

Two consequences of that arrangement bear on the node API. It is consumed both
directly, by a caller holding it across an FFI boundary, and indirectly, through
the JSON-RPC proxy and the client library, so its surface has to survive
projection onto a wire protocol rather than assuming a local caller. And because
a consumer may reach the node through either path, the two must express the same
semantics.

## Unified Logos Development Kit & artefact size

The architecture proposed here assumes a similar development kit will be needed
for Logos Blockchain, and for Logos Delivery, Chat and Storage in turn,
depending on the appetite for integrating those protocols into existing
applications.

A unified Logos Development Kit would simplify integration for a developer, at
the cost of artefact size. BDK is the example to reason from: it publishes a
prebuilt AAR, `bdk-android`, whose `libbdkffi.so` carries every capability the
kit offers. A single `.so` for all of Logos may be too large for mobile, where
Android and iOS both impose size limits past which the experience degrades for
users and developers alike. Publishing several variants of a unified kit may in
turn cost more to maintain than keeping the kits separate in the first place.

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
of its own. That is also before the four remaining Logos protocols, and before
any per-architecture packaging.

Neither number above is for a mobile target, and neither can be today: the ZK
circuit artefacts both libraries depend on are published for Linux, macOS and
Windows only, so an Android or iOS build fails before it produces a size.
Building those circuits for mobile is a prerequisite to answering the question,
and a proposal is expected to obtain the figures rather than assume them.

#### Different prebuilt artefacts per platform

One candidate strategy, offered as a starting point rather than a requirement,
is to let the published artefacts differ by platform. A desktop or server
artefact carries the embedded node; a mobile artefact omits it and reaches a
remote node through the client instead. The kit is then the same source with the
same surface, and only what is shipped varies, which keeps the choice away from
the integrator: a developer consuming a prebuilt library cannot set a build
feature, so the platform an artefact targets has to decide what is in it.
Prebuilt artefacts are still worth publishing for every platform, including the
desktop and server ones that carry the node, since a build from source over this
dependency graph is not a reasonable first step for an integrator, and a server
integrator in Go or on the JVM may hold no Rust toolchain at all.

The cost of that strategy is a wider artefact matrix, and one question it leaves
open is what the mobile artefact does about the node surface it does not carry:
omitting those types keeps the artefact honest at the price of two shapes for
one language binding, while exposing them and failing at run time keeps one
shape at the price of a surprise. A proposal states which it chose.

## The bitcoind-style JSON-RPC node wallet

Historically geth, and still today bitcoind, carry wallet features and a wallet
API inside the node. The ecosystem has moved away from that arrangement
([Appendix: Wallet Libraries Ecosystem, section 1](./wallet-libraries-ecosystem.md#1-where-wallet-functionality-lives)):
geth removed the `personal` namespace, Bitcoin Core is separating its wallet
into its own process, and chains newer than those two ship wallet libraries
without ever putting a wallet in the node.

LEZ is nonetheless positioned to offer that shape cheaply, because the
`lez_core` module already carries the wallet beside the node. One further
component, the JSON-RPC proxy module of deliverable 3, would expose the wallet
API over the same transport as the node API, which makes wallet integration
reachable from a server or cloud environment, and potentially a desktop one,
alongside the Logos Core and Basecamp path.

This could be a quicker and cheaper way to provide wallet integration for both
LEZ and Logos Blockchain. The demand for it would need to be validated first,
however, given the move away from it the ecosystem has demonstrated.

## Logos stack readiness

Two items bear on the architecture proposed above.

**Basecamp mobile readiness.** The first diagram is marked as applying to any
Basecamp application, but mobile support is not planned for testnet 0.3, so at
the time of writing that shape is reachable on desktop only.

**The target topology for LEZ and Logos Blockchain on mobile.** It is not yet
settled whether a mobile device is expected to run light versions of the chain
nodes and join the peer-to-peer networks directly, to reach remote nodes over
RPC alone, or to combine the two. That decision governs the unified development
kit and what embedding a node means per platform, so the sizes and the
per-platform artefact strategy above are provisional until it is made.

## The LEZ node API is the only API for LEZ chain state access

The LEZ node API is the whole of the surface available to a consumer, in both
directions. A LEZ node is a black box: what it is made of, which part of it
answers a given call, and how those parts talk to each other are implementation,
not API. The API says what a consumer may ask for and what comes back, and
nothing about how a node arranges itself to answer.

The node serves every read. Where it does not already hold what a consumer asks
for, it obtains it from wherever that data lives rather than directing the
consumer elsewhere. That internal arrangement is free to change without a
consumer being able to tell.

It is also the only way onto the chain. A signed transaction, and the new
commitments that come with a privacy-preserving one, are produced by the LEZ
wallet and reach the network through this API, which relays what it is handed
without constructing or signing anything itself. A consumer that holds keys
still writes through the node, so the wallet and the node divide the work rather
than offering two routes to the chain.

A capability a consumer needs is therefore required of the node regardless of
which part of it holds the data today. The pending set is the clearest case, and
the same reasoning governs every read and write RFP-027 requires.
