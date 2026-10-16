# Appendix: L1 SDK Strategies

This appendix surveys what SDKs recent Layer 1 blockchains shipped, in which
languages, and how those SDKs relate to each other: whether a chain maintains a
native implementation per language, or generates bindings from one core library.
It bears directly on the LEZ-DK, which follows the Bitcoin Development Kit in
generating per-language libraries from a single Rust core.

The companion appendices cover adjacent concerns.
[Appendix: Wallet Libraries and Key Management](./wallet-libraries-ecosystem.md)
surveys where wallet functionality lives per chain, and
[Appendix: Wallet Application Technology Stacks](./wallet-tech-stacks.md)
surveys what wallet applications are built with. This one is about the libraries
a chain publishes for integrators.

> **Sourcing note.** This appendix is drafted from secondary material and is
> **pending fact-check**. It carries no references yet. Unlike the sourced
> appendices in this directory, no claim here has been confirmed against a
> first-party repository or document, and the SDK inventories in particular are
> the kind of list that goes stale quickly. Treat it as a hypothesis to verify.

## 1. What recent L1s shipped

### 1.1 Sui, launched 2023

The most comprehensive suite of the set, non-EVM and therefore unable to inherit
Ethereum tooling.

| SDK                 | Language          | Purpose                              |
| ------------------- | ----------------- | ------------------------------------ |
| Sui TypeScript SDK  | TypeScript        | Core chain interactions              |
| Sui dApp Kit        | TypeScript, React | Components, hooks, wallet connection |
| Sui Wallet Standard | TypeScript        | Wallet interoperability              |
| Sui Rust SDK        | Rust              | Native performance                   |
| Pysui               | Python            | Python-native development            |
| SuiKit              | Swift             | iOS and macOS                        |
| Sui Go SDK          | Go                | Backend infrastructure               |
| Sui Kotlin SDK      | Kotlin            | Android                              |
| Sui Dart SDK        | Dart              | Flutter                              |
| Sui Unity SDK       | C#                | Games                                |

Alongside those, four wallet-specific tools: zkLogin for passwordless
authentication against Web2 credentials, Enoki for sponsored transactions and
embedded wallets, zkSend for link-based transfers, and the Slush embedded wallet
SDK.

### 1.2 Aptos, launched 2022

Also non-EVM, also multi-language from the start, though a narrower set than
Sui's: TypeScript (the recommended entry point), Python, Go and Rust, plus the
Aptos Wallet Adapter as a wallet-connection layer and Forklift for testing and
scripting against a forked network.

### 1.3 Sei, launched 2023, V2 in 2024

A hybrid. Sei publishes `sei-js` for chain-specific features, `create-sei` for
project scaffolding, the Sei Global Wallet for onboarding users arriving from
other chains, and AI-oriented developer tooling. Because Sei V2 is
EVM-compatible, it leans on standard EVM tooling for the rest rather than
building wallet infrastructure of its own.

### 1.4 Berachain, launched 2025

EVM-compatible, and correspondingly thin: no native wallet SDK, standard EVM
wallets (MetaMask, Rabby, Frame, Rainbow, Safe), thirdweb for wallet connection,
and account-abstraction SDKs from Dynamic, Privy, Particle, Turnkey and ZeroDev.
The one first-party library, Berancer SDK, targets its own exchange protocol
rather than the chain.

### 1.5 Monad, testnet live, mainnet pending

The same EVM pattern: no native wallet SDK, existing EVM wallets, `ethers.js`,
`viem` and `web3.js`, with Web3Auth for embedded wallets.

### 1.6 Injective, launched 2021

Cosmos SDK-based, publishing TypeScript, Python and Go SDKs, an AI agent toolkit
in TypeScript, and a Rust CLI for wallet management and transaction preparation.

## 2. The pattern by chain family

Three groups, and which group a chain falls into predicts its SDK strategy
better than its launch date does.

**Non-EVM chains** build comprehensive native SDKs across many languages from
day one, because there is no existing tooling to inherit. Sui and Aptos are the
clear cases.

**EVM-compatible chains** defer wallet SDKs almost entirely and rely on the
Ethereum ecosystem. Monad, Berachain and Sei V2 all do this, publishing
chain-specific libraries only where they have chain-specific features.

**Cosmos SDK chains** use Cosmos ecosystem tooling and add TypeScript, Python
and Go SDKs for their own features. Injective, Celestia and Sei V1 follow this.

Across all three, wallet abstraction layers (Dynamic, Privy, Particle) are
becoming a standard integration, letting a new L1 offer embedded wallets without
building them.

## 3. Does any recent L1 use the BDK pattern?

This is the question that bears on the LEZ-DK, and the answer appears to be no.

**None of Sui, Aptos, Sei, Berachain, Monad, Injective or Celestia generates its
per-language SDKs from a single core library.** Sui and Aptos both have Rust
SDKs, but their other SDKs are independent native implementations that reach the
chain over JSON-RPC rather than bindings over a shared core. Sui's TypeScript
SDK is built on the Sui JSON-RPC API; Aptos's TypeScript SDK is a native
TypeScript library. Each language is a separate codebase.

### 3.1 Who does use it

The pattern exists, but outside the L1 launch cohort:

- **Bitcoin Development Kit**, which pioneered it: a Rust core, with `bdk-ffi`
  using Mozilla's UniFFI to generate Kotlin, Swift and Python bindings from one
  codebase.
- **Crypto.com's DeFi Wallet** (`defi-wallet-core-rs`): a Rust core for Cosmos
  SDK and Ethereum chains, with UniFFI generating Kotlin and Swift bindings,
  supporting Cronos and Crypto.org Chain among others.

### 3.2 Why L1s skip it

Four reasons are advanced, and they are worth weighing separately because not
all of them apply to every case:

- **RPC-first architecture.** A chain exposing a rich JSON-RPC or GraphQL
  surface makes a native SDK per language straightforward, since each one is a
  client for a wire protocol rather than a wrapper over logic.
- **Idiomatic developer experience.** A native SDK can follow the conventions of
  its language without the constraints an FFI boundary imposes.
- **Web dominance.** TypeScript and browser environments cannot use FFI
  directly, so the largest single audience needs WASM or a pure-JavaScript
  implementation regardless.
- **VM-specific transaction building.** Sui and Aptos use the Move VM, and
  building a transaction requires Move-specific logic that is reportedly easier
  to implement per language than to project across a binding layer.

**Solana is the same shape as Sui.** Its `@solana/web3.js` TypeScript SDK is a
native TypeScript reimplementation (~30,000 lines), and its Rust `solana-sdk` is
a separate native Rust library. There is no UniFFI or Rust-FFI binding layer
for the main SDKs. Solana, like Sui, can do this because a Solana client only
needs to assemble transaction bytes, sign them with ed25519, and submit them to
an RPC node. The validator executes the program logic; the wallet does not.

### 3.3 Where the split falls

The distinction that emerges is between wallet libraries and full chain SDKs.

The BDK pattern suits **wallet** functionality, where the work is cryptographic,
the logic is substantial, and a single audited core is a security asset:
signing, descriptor management, PSBT handling.

Native per-language implementations remain dominant for **full chain SDKs**,
where the work is transaction building, VM-specific compilation and state
management, and much of the value is in being idiomatic.

That split is the relevant one for the LEZ-DK, which carries both a wallet and a
node client in one kit.

## 4. Quantifying the client-side complexity gap

The previous sections describe the pattern qualitatively. This section puts
numbers on it by comparing the **Rust SDKs** of Bitcoin (`rust-bitcoin`), Sui
(`sui-rust-sdk`), and Solana (`solana-sdk` plus `solana-rpc-client`). Comparing
Rust-to-Rust removes the language-porting variable and isolates how much logic
each chain actually requires a client to run locally.

> **Sourcing note for the numbers below.** The line counts were produced by
> cloning the upstream repositories and counting `.rs` files. They are raw LOC,
> not complexity-adjusted estimates, and should be read as order-of-magnitude
> indicators rather than exact claims. The repositories analyzed were:
> `rust-bitcoin/rust-bitcoin`, `MystenLabs/sui-rust-sdk`,
> `anza-xyz/solana-sdk`, and `anza-xyz/agave` (for `solana-rpc-client`).

### 4.1 Total SDK size

| Metric | `rust-bitcoin` | `sui-rust-sdk` | `solana-sdk` + `rpc-client` |
| --- | --- | --- | --- |
| Total Rust LOC | ~99,000 | ~168,000* | ~88,000** |
| Crates | 21 | 10 | 120 |
| Unsafe-block files | 21 | 3 | 84 |

\* `sui-rust-sdk`'s total is inflated by ~107,000 LOC of auto-generated
protobuf, gRPC and GraphQL binding code in the `sui-rpc` crate. The hand-written
logic is roughly 60,000 LOC.

\*\* `solana-sdk`'s total includes many crates for **on-chain programs**
(`program`, `program-entrypoint`, `cpi`, etc.). The client-facing subset is
roughly 30,000 LOC of types and helpers plus ~14,000 LOC of RPC client.

### 4.2 Domain-critical client logic

Total LOC is misleading. The relevant question is: how much code must run in
the client (wallet, SDK) to produce a valid transaction? The table below
isolates that work.

| Concern | `rust-bitcoin` | `sui-rust-sdk` | `solana-sdk` + `rpc-client` |
| --- | --- | --- | --- |
| Script / VM execution | ~3,700 LOC | 0 — validators run Move | 0 — validators run BPF/SVM |
| Sighash / signing hash construction | ~1,900 LOC | ~200 LOC (intent signing) | ~500 LOC (message hash) |
| Locktime / timelock validation | ~3,200 LOC | 0 — on-chain | 0 — on-chain |
| Taproot / witness handling | ~4,400 LOC | 0 | 0 |
| Address / descriptor derivation | ~2,500 LOC | ~300 LOC (simple hash) | ~300 LOC (pubkey-based) |
| Transaction building | ~5,000+ LOC (coin selection, PSBT, fee estimation) | ~2,000 LOC | ~2,300 LOC (message/tx) |
| UTXO / local state tracking | ~8,000+ LOC | 0 — query node | 0 — query node |
| RPC client | N/A (out of scope) | ~107,000 LOC (auto-generated) | ~14,000 LOC (hand-written) |
| **Total domain-critical client LOC** | **~25,000 LOC** | **~2,500 LOC** | **~3,100 LOC** |

### 4.3 What the ratio means

Bitcoin requires roughly **10× more client-side consensus-critical logic** than
Sui or Solana. That logic is not incidental; it is the work a Bitcoin wallet
must do because Bitcoin pushes spending conditions, state tracking and
validation to the wallet:

- A Bitcoin wallet must interpret the script that locks a UTXO to know whether
  it can spend it.
- It must construct a valid witness (signatures, hash preimages, multisig
  satisfactions) that satisfies the script.
- It must track the global UTXO set to know which coins belong to it.
- It must compute transaction hashes for signing using intricate BIP-143 and
  BIP-341 rules.

Sui and Solana wallets do none of this. They assemble transaction bytes, sign
them, and hand them to a validator. The validator executes Move or BPF/SVM code
and returns the result. The client-side surface is small enough that native
reimplementations per language are practical.

### 4.4 Why this justifies the BDK pattern

The BDK pattern — a single Rust core with UniFFI-generated bindings — is a
response to that 10× gap. Reimplementing 25,000 lines of script validation,
sighash construction, descriptor satisfaction and UTXO management in Kotlin,
Swift and Python would require:

- Deep Bitcoin protocol expertise in each language team.
- Independent security audits for each reimplementation.
- Years of feature-parity lag, because new BIPs land in Rust first.

By centralizing the complex logic in one Rust core and projecting bindings over
it, BDK trades API idiomaticity for correctness and security. Sui and Solana do
not face that tradeoff because their clients are thin RPC wrappers with modest
local logic.

For the LEZ-DK, the relevant question is therefore: which side of the split
will the Logos SDK sit on? If it resembles a Bitcoin wallet — local state,
script validation, complex signing — the BDK/UniFFI pattern is strongly
justified. If it resembles a Sui or Solana client — serialize, sign, submit —
native per-language SDKs are the simpler and more ergonomic choice.

## 5. Where the Logos repositories sit

The same framework can be applied to the Logos codebases themselves. The
question is whether each repository puts the wallet complexity on the client
side (BDK-like) or on the node side (Sui/Solana-like), and whether the
per-language boundary is an FFI into a wallet core or an HTTP/RPC call into a
node service.

> **Sourcing note.** The figures below are raw `.rs` LOC counts from the local
> working copies at `~/src/logos-blockchain/logos-execution-zone` and
> `~/src/logos-blockchain/logos-blockchain`. They are order-of-magnitude
> indicators, not complexity-adjusted estimates.

### 5.1 `logos-execution-zone`

This repository fits the BDK gap. It contains a dedicated `lez/wallet` crate
and a dedicated `lez/wallet-ffi` crate that exposes it as a `cdylib` and
`staticlib`.

| Component | LOC | Role |
| --- | --- | --- |
| `lez/wallet` | ~12,800 | Client-side wallet logic: accounts, sync, program facades, local storage |
| `lez/wallet-ffi` | ~5,100 | FFI bindings over the wallet crate |
| `lez/sequencer` | ~18,500 | Node-side transaction ordering |
| `lez/mempool` | ~174 | Node-side mempool |
| `lez/storage` | ~6,400 | Node-side storage |
| `lez/chain_state` | ~2,400 | Node-side chain state |
| `lez/indexer` | ~12,500 | Node-side indexer |
| **Total repository** | **~122,000** | |

The wallet and node code are cleanly separated, and the wallet crate is large
enough that reimplementing it per target language would be expensive and
security-sensitive. Using UniFFI or a similar binding generator over the Rust
wallet core is the natural choice here.

### 5.2 `logos-blockchain`

This repository does **not** fit the same gap. Its wallet surface is small and
exposed from the node, not from a client-side core.

| Component | LOC | Role |
| --- | --- | --- |
| `wallet` | ~2,500 | Wallet state types and helpers (could run client-side, but is thin) |
| `services/wallet` | ~2,700 | Node-side wallet service that wraps the wallet crate |
| `wallet-http-client` | ~115 | Thin HTTP client that calls the node-side wallet service |
| `c-bindings` | ~5,300 | C bindings over the **entire Logos Blockchain node**, not just the wallet |
| `services/chain` | ~14,500 | Node-side chain service |
| `blend` | ~20,200 | Node-side mix-network / blend service |
| `core` | ~18,800 | Core types and logic |
| `ledger` | ~12,700 | Node-side ledger |
| **Total repository** | **~227,000** | |

The `c-bindings` crate is the clearest signal: it exposes `LogosBlockchainNode`
and APIs for consensus, chain, blend, storage, leader election, and wallet
operations. A client using these bindings is not calling a wallet library; it
is driving a full node. The wallet operations inside `c-bindings/src/api/wallet.rs`
are routed through the node-side wallet service.

### 5.3 What the boundary means for SDK strategy

`logos-execution-zone` and `logos-blockchain` therefore call for different
integration patterns:

- **`logos-execution-zone`**: A Rust wallet core with FFI-generated bindings
  (the BDK pattern) is appropriate because the wallet logic is substantial and
  client-side.

- **`logos-blockchain`**: The natural client boundary is the HTTP API exposed
  by `services/api` and consumed by `wallet-http-client`, plus the full-node
  C bindings for embedded or native use cases. A native SDK per language is
  practical because the client only needs to serialize requests, sign, and submit
  to the node.

This split mirrors the Bitcoin vs. Sui/Solana distinction: the more work the
wallet must do locally (UTXO tracking, state application, proof generation), the
stronger the case for a shared Rust core with FFI bindings. The more work the
node does on the wallet's behalf, the more an RPC-first SDK strategy suffices.

## 6. The Logos module system: a third pattern

There is a third relevant pattern in the Logos stack that is easy to confuse
with the BDK/UniFFI approach because it also involves Rust and FFI, but it is
structurally different. This section clarifies it by examining
`logos-rust-sdk` and the `logos-liblogos` module runtime.

> **Sourcing note.** `logos-rust-sdk`, `logos-js-sdk` and `logos-liblogos` were
> cloned from `github.com/logos-co`.

### 6.1 What `logos-rust-sdk` is not

`logos-rust-sdk` is **not** a BDK-style library that an application links and
calls into. From its `Cargo.toml`:

```toml
[lib]
crate-type = ["rlib"]
```

and its `build.rs`:

> *"The `lp_*` symbols are intentionally left unresolved at Rust compile time
> (rlib output). They are satisfied at final link time: for a module plugin,
> against the logos-protocol archive embedded in the plugin."*

In other words, this crate is a **runtime for Rust-authored Logos modules**.
It lets a module written in Rust call other Logos modules over the `lp_*` C
ABI. The consumer is a module, not an end-user application.

### 6.2 How modules are brought in

The module architecture is defined by `logos-liblogos`, the core runtime
library. The relevant pieces are split across repositories so each can be
swapped:

| Component | Repository | Responsibility |
| --- | --- | --- |
| `logos-liblogos` | `logos-co/logos-liblogos` | Core runtime: `liblogos_core` C API + module registry |
| `logos-container` | `logos-co/logos-container` | Container contract (where/how a module runs) |
| `logos-container-subprocess` | `logos-co/logos-container-subprocess` | Default container: one OS process per module |
| `logos-module-loader` | `logos-co/logos-module-loader` | Format-loader contract (what kind of module) |
| `logos-module-loader-qt` | `logos-co/logos-module-loader-qt` | Default loader: Qt plugin format + `logos_host_qt` binary |
| `logos-protocol` | `logos-co/logos-protocol` | Language-neutral `lp_*` C ABI |
| `logos-rust-sdk` | `logos-co/logos-rust-sdk` | Rust module runtime |
| `logos-cpp-sdk` | `logos-co/logos-cpp-sdk` | C++ module runtime |
| `logos-nim-sdk` / `liblogos` | `logos-co/logos-nim-sdk` | Nim module runtime |
| `logos-js-sdk` | `logos-co/logos-js-sdk` | JavaScript module runtime (Qt-free, via koffi) |

The loading flow is:

1. A module is packaged as a `.lgx` archive or built as a Qt plugin
   (`.so`/`.dylib`/`.dll`) with a `metadata.json`.
2. `logos-liblogos` discovers it in a module directory.
3. On `logos_core_load_module(name, ...)`, the core resolves dependencies,
   picks a `ModuleLoader` from the registry, and asks it to load.
4. The default `CompositeModuleLoader` pairs `SubprocessContainer` with
   `QtPluginFormatLoader`:
   - The format loader resolves the `logos_host_qt` host binary and builds CLI
     arguments.
   - The container spawns a new OS process running `logos_host_qt <module>`.
5. The core generates a UUID auth token and sends it to the child process over
   the container's private channel (stdin pipe, by default).
6. The host process loads the plugin, validates its declared name, and registers
   it with the remote object registry.
7. Modules call each other via typed RPC over the `lp_*` ABI, using language
   SDKs (`logos-rust-sdk`, `logos-cpp-sdk`, etc.).

So modules are **pre-compiled plugins loaded into separate processes**, not
libraries linked into a single application process.

### 6.3 Why this is different from BDK

| | **BDK (`bdk-ffi`)** | **Logos module system** |
| --- | --- | --- |
| **Unit of reuse** | Single Rust library | Independent module plugins |
| **Process model** | Same process as the app | Separate process per module |
| **Binding boundary** | App → Rust library via FFI | Module → module via `lp_*` C ABI over IPC |
| **Compilation artifact** | `rlib`/`cdylib` consumed directly by app | Qt plugin / `.lgx` consumed by module host |
| **Lifecycle** | App initializes and calls functions | Host spawns, tokens, unloads modules |
| **Sandboxing** | None (library runs in app) | Process isolation by default |
| **Use case** | Wallet logic shared across languages | Modular dApp composed of sandboxed modules |

BDK answers the question: *"How do I call one wallet library from many
languages?"* The Logos module system answers a different question: *"How do I
compose a decentralized application from independently developed, sandboxed
modules?"*

### 6.4 What `logos-js-sdk` looks like

`logos-js-sdk` is the JavaScript member of the same family. It is small
(~870 LOC) and Qt-free:

- It loads `liblogos_protocol.{so,dylib}` directly via
  [koffi](https://koffi.dev), a Node.js FFI library.
- It binds the same `lp_*` C ABI as `logos-rust-sdk`.
- It does **not** use `liblogos_core` and does **not** embed a Qt host or event
  loop.
- A Node process can act as a **consumer** (`LogosClient`) calling other modules,
  or as a **provider** (`Provider`) serving methods and events to other modules.
- Communication is over plain transports: TCP, TCP+SSL, or a plain-local Unix
  socket.

Example consumer:

```js
const { LogosClient, tcp } = require('logos-js-sdk');
const logos = new LogosClient('my_app', { transport: tcp('127.0.0.1', 6001) });
const calc = logos.module('calc_module');
const sum = await calc.call('add', 5, 3);
```

Example provider:

```js
const { Provider, tcp } = require('logos-js-sdk');
const p = new Provider('greeter', tcp('127.0.0.1', 6002));
p.register({
  handlers: { hello: (name) => `hi ${name}` },
  events: ['greeted'],
});
p.emit('greeted', 'world');
```

This is still not the BDK pattern. The JS code is not calling a wallet library
via FFI; it is loading a protocol library and participating in the Logos module
network as an out-of-process peer. The provider half is currently gated on a
`logos-protocol` feature branch (`logos-protocol#12`); the consumer half works
against protocol master.

### 6.5 Implication for the LEZ-DK

The LEZ-DK appears to follow the BDK pattern (Section 5.1): a Rust wallet core
with FFI-generated bindings. It should not be confused with the Logos module
system, even though both use Rust and FFI. The module system is relevant only if
the LEZ-DK is intended to run *as a Logos module* — in which case the
per-language surface would be generated module clients, not a wallet FFI — or if
the LEZ-DK needs to *host* Logos modules, in which case the integration boundary
is the `logos-liblogos` C API, not UniFFI.
