## Proposed architecture

Planning has moved on since this issue was written. The work now splits four
ways:

1. FFI API for the LEZ **indexer**: the equivalent of a node API, in the sense
   the `eth` namespace is on Ethereum's JSON-RPC.
2. FFI API for the LEZ **wallet**: the equivalent of what ethers.js provides,
   meaning key handling, derivation, and signing.
3. The **JSON-RPC proxy**, for consumers that want a wire protocol rather
   than a linked library.
4. The **SDK**, which consumes both FFIs.

This issue is items 3 and 4. Item 1 is #219, now scoped to the indexer FFI.
Item 2 has no issue yet.

## Why the SDK stays thin, and what it links against

The intent is not to rewrite the LEZ wallet functions in TypeScript. They stay
in Rust, and the TypeScript library remains as light as possible.

The shape to follow is BDK's. `bdk-ffi` is a library a consumer links, not a
service a consumer runs: there is no plugin lifecycle, no process to supervise,
and no inter-process boundary. Kotlin, Swift, Python and React Native packages
each embed the same Rust core through uniffi-rs and call it in process.

That is a different arrangement from the LEZ core module today, which is a
Logos Core plugin with `open` and `save` lifecycle calls and a held wallet
handle. Under the BDK shape the wallet FFI is something an application links
and calls, so an integrator does not run a module at all, and the packaging
problem reduces to shipping the right binary per platform rather than
supervising a process.

This also resolves what would otherwise be an open question on this issue,
whether the SDK is an API client or a wallet library. It is one deliverable
consuming two FFIs, and the split sits at the FFI layer rather than inside the
SDK.

## Two notes from the ecosystem survey

**The custody argument does not apply here.** Chains have moved wallet
functionality out of nodes: geth removed the `personal` namespace in 2024, and
six of eight surveyed chains put wallet functionality in a library rather than
the node. The reason is that a node is a network daemon that should not hold
user keys. A linked wallet library is not that, which is why every one of those
six chains still has one. The direction here is the same: keys sit in a library
the application links, not in anything running on the network.

Generated bindings are the intended path, following BDK, so the FFI surface is
constrained by what the binding generator can express. That is a real design
input rather than an afterthought: a surface designed for uniffi or an
equivalent looks different from one designed only for a hand-written C++
wrapper.

**LEZ can support watch-only derivation, and should decide whether to.** LEZ
signs with BIP-340 Schnorr over secp256k1 (`k256::schnorr` in
`lee/state_machine/src/signature/mod.rs`). That matters because watch-only
deposit-address derivation, the capability a centralised exchange needs most,
works on secp256k1 through public derivation and is impossible on ed25519
chains such as Solana, Stellar, NEAR, and Sui, which permit only hardened
derivation. LEZ is in the same family as Bitcoin, Ethereum, and Cosmos here.

The current wallet FFI does not expose it. There is no derivation-path
function, no extended-public-key export, and `wallet_ffi_restore_data` takes a
`depth` parameter whose documentation warns of "exponential growth in execution
time", which reads as scanning rather than deterministic derivation. Private
accounts separately carry a viewing key (`vpk`), which is the privacy-chain
equivalent and is also unexposed. Both belong in the item 2 discussion.

## Source material

The Source material section cites `logos-api-sdk-research`. That repository is
AI generated and is not a usable reference; spot-checking found figures its own
lockfile contradicts. Three replacement appendices are in review on the `rfp/blockchain-api` branch
of `logos-co/rfp`, written from first-party sources with every URL fetched and
every quotation verified against the original, and reviewed by independent
adversarial passes against the codebases:

- `appendix/blockchain-api-sdk-ecosystem.md`, covering 34 API functions across
  eight chains, transports, SDK languages, response shapes, and documented
  origin history.
- `appendix/logos-api-surfaces.md`, an as-built inventory of the LEZ indexer
  FFI and RPC, sequencer RPC, wallet FFI and `lez_core` module, and the L1
  bindings, routes, and module.
- `appendix/wallet-libraries-ecosystem.md`, covering where wallet functionality
  lives per chain, key derivation standards, watch-only support, and hardware
  and offline signing. This is the direct input for item 2.

## On the linked-library dependency

Following BDK means a consuming application links a native library rather than
running a module. That narrows the cost considerably: it is a build and
packaging concern (ship the right binary per platform, and per architecture)
rather than a runtime one, and it removes the process supervision, the
lifecycle calls, and the inter-process boundary the current core module has.

The residual constraint is the browser, where a linked native library is not
available and WebAssembly is the only route. BDK has the same constraint, so
the question of whether the Rust core compiles to wasm, and what it costs if it
does, is worth answering early rather than discovering later.

The longer-term position is unchanged and worth stating: this is a staging
decision, not a permanent shape. The dependency exists because the wallet logic
is written once in Rust rather than per language. Anyone who wants to remove it
can produce a native implementation in their target language, and their users
then link nothing. That needs no coordination or permission, since the FFI is
the contract and a conformant reimplementation is a drop-in substitute. It is
how bitcoinjs-lib came about, without Bitcoin Core's involvement.

BDK is worth studying for one further reason. It sits on top of rust-bitcoin
rather than replacing it: rust-bitcoin owns the primitives, keys, addresses,
script, transaction encoding, while BDK owns the wallet logic, descriptors,
coin selection, chain sync, and PSBT flows. The equivalent split here is that
`lee/state_machine` holds the primitives and the wallet FFI is the layer above
it, which suggests the FFI boundary should be drawn at wallet semantics rather
than at whatever the current module happens to expose.
