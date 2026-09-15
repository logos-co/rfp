# Blockchain API and SDK architecture

Six pieces. Items 1 and 2 define surfaces; 3 to 6 consume them.

## 1. Indexer FFI

The equivalent of a node API, in the sense the `eth` namespace is on Ethereum's
JSON-RPC. The indexer is expected to run as a node, in the shape an RPC provider
runs one.

Issue [#219](https://github.com/logos-co/ecosystem/issues/219). Draft at
`RFPs/RFP-021-lez-indexer-ffi.md`.

## 2. Wallet FFI

The equivalent of what ethers.js provides: key handling, derivation, signing.

Two deployment shapes, and both are wanted:

- Inside Basecamp, the wallet **could** run as a node or binary.
- Outside Basecamp, follow BDK: ship the wallet as a library in several
  languages, generated from one Rust core over an FFI.

No issue yet.

## 3. JSON-RPC proxy module

Can cover both wallet and indexer functionality. The indexer half is the more
critical of the two.

Part of issue [#220](https://github.com/logos-co/ecosystem/issues/220).

## 4. Wallet SDK

BDK-shaped: a library per language over the Rust FFI, not a client for a wire
protocol. Target languages depend on the stack and need prioritising with BD.

## 5. Indexer SDK

A library for hitting the indexer's JSON-RPC surface. The wallet SDK may use it
to reach a running indexer over JSON-RPC.

Rust is required. Other languages depend on BD.

## 6. Other proxy modules

Further transport bindings beyond JSON-RPC: gRPC, GraphQL, Mesh or Rosetta.
Issue [#222](https://github.com/logos-co/ecosystem/issues/222).

Ecosystem adoption from
[Appendix: Blockchain API and SDK Ecosystem](../appendix/blockchain-api-sdk-ecosystem.md),
section 2: gRPC 9 of 22, GraphQL 6 of 22, and REST 19 of 22 with OpenAPI 16 of
22. Institutional adapter surfaces are almost entirely REST with OpenAPI, which
is the Mesh and Rosetta lineage: seven first-party adapters exist across Aptos,
Hedera, Mina, NEAR, Sui, ICP, and Cardano.

## Language candidates for items 4 and 5

None of this is evidenced as a ranking of integrator demand. The ecosystem
survey records which SDKs chains publish, not which languages partners ask for,
and the point is worth keeping separate from the counts below.

### Centralised exchange integration

| Language | Ecosystem presence | Note |
| --- | --- | --- |
| Go | 7 of 8, first party on 4 | Common in exchange backends |
| Java or Kotlin | 6 of 8 | XRPL ships first-party `xrpl4j`, framed around institutional integrators |
| TypeScript on Node | 8 of 8, first party on 6 | Often operations tooling rather than the settlement path |
| Python | 7 of 8 | Reconciliation and analytics |
| Rust | 6 of 8 | For an integrator linking the core directly |

### Wallet integration

| Target | Language | BDK precedent |
| --- | --- | --- |
| Android | Kotlin | `bdk-android`, built in-repo |
| iOS and macOS | Swift | `bdk-swift`, built in-repo |
| Cross-platform mobile and desktop | Dart and Flutter | `bdk-dart`, downstream repository |
| React Native | TypeScript | `bdk-rn`, downstream repository |
| Server-side JVM | Kotlin | `bdk-jvm`, downstream repository |

Cake Wallet, as a desktop and mobile reference point, is 97 per cent Dart on
Flutter, so Dart is a real target rather than a hypothetical one.

### What the binding mechanism decides

uniffi, which BDK uses, generates Kotlin, Swift, Python, and Ruby. It does not
generate Go, and it does not generate TypeScript for Node. That is why `bdk-rn`
is a separate downstream project and why BDK has no Go binding at all.

So the two languages a centralised exchange is most likely to want, Go and Node,
are the two the binding generator will not produce. Three ways to close that:

- A C ABI with hand-written cgo and napi bindings.
- WebAssembly, which covers Node and the browser from one artefact, at the cost
  of filesystem access, threads, and the performance of local proving.
- The JSON-RPC proxy, item 3, with an SDK per language on top, which is item 5.

That last route is why item 3 is more than a convenience: for languages the FFI
cannot reach, it is the integration path.

## How BDK arranges this, for reference

BDK does not talk to Bitcoin nodes itself. It ships three optional adapter
crates, each with a different transport:

| Crate | Backend | Transport |
| --- | --- | --- |
| `bdk_electrum` | Electrum server | JSON-RPC over TCP or TLS |
| `bdk_esplora` | Esplora | REST over HTTP |
| `bdk_bitcoind_rpc` | `bitcoind` | JSON-RPC over HTTP |

The wallet core knows none of them. It defines what it needs in Rust types and
each crate adapts a source into that shape, which is why three different
protocols can sit underneath one wallet.

Two of the three backends are indexers rather than nodes, because `bitcoind`
alone cannot answer which transactions touch an address. That supports treating
the indexer as the integration surface here.

BDK also sits on top of rust-bitcoin rather than replacing it: rust-bitcoin owns
primitives, keys, addresses, script, and transaction encoding, while BDK owns
wallet logic, descriptors, coin selection, chain sync, and PSBT flows. The
equivalent split is that `lee/state_machine` holds the primitives and the wallet
FFI is the layer above.

## Open question

If the wallet SDK reaches the indexer through the indexer SDK over JSON-RPC
(item 5), the wallet is transport-coupled the way BDK's adapters are, and an
alternative data source is possible. If the wallet FFI instead calls the indexer
FFI directly in process, that is tighter than anything BDK does, and the two
ship and version together. Worth deciding deliberately rather than by default.
