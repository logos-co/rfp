# Appendix: Wallet Libraries and Key Management

This appendix surveys where wallet functionality lives across blockchain
ecosystems: key generation, hierarchical derivation, deposit-address generation,
and transaction signing. It is a separate concern from the node API, which is
surveyed in
[Appendix: Blockchain API and SDK Ecosystem](./blockchain-api-sdk-ecosystem.md).
A node API answers what a client can read and submit; a wallet library answers
how a client comes to hold keys and produce signed bytes at all, much of which
never touches a node.

The distinction matters for integrators. An exchange or custodian listing a
chain needs deposit-address derivation and offline signing before it needs any
read method, and the question it must answer first is whether the chain's own
software provides those or whether it must adopt a third-party library.

Every claim carries a first-party source that was fetched and confirmed to
resolve. Where something could not be confirmed, the entry reads `[NOT FOUND]`,
which records the limit of the sourcing rather than an assertion of absence.

## Chains surveyed

The same eight chains as the node API survey, in the same order.

| Chain    | Node wallet           | Canonical library                    | First party |
| -------- | --------------------- | ------------------------------------ | ----------- |
| Ethereum | removed from geth [1] | ethers.js [2]                        | no          |
| Bitcoin  | yes, optional [3]     | rust-bitcoin, BDK, bitcoinjs-lib [4] | no          |
| Solana   | never                 | `solana-sdk`, `kit` [5]              | yes         |
| XRPL     | admin only [6]        | xrpl.js [7]                          | yes         |
| Cosmos   | never over RPC [8]    | CosmJS [9]                           | yes         |
| Stellar  | never                 | js-stellar-sdk [10]                  | yes         |
| NEAR     | never                 | near-api-js [11]                     | yes         |
| Sui      | never                 | `@mysten/sui` [12]                   | yes         |

Six of the eight put wallet functionality in a library rather than the node. The
two that do not are the two largest, and they are also the two with no
first-party library: an integrator on Bitcoin or Ethereum adopts community code,
while every newer chain ships its own.

## 1. Where wallet functionality lives

### 1.1 Bitcoin keeps it in the node

Bitcoin Core still ships `createwallet`, `getnewaddress`,
`signrawtransactionwithwallet`, and `listunspent` [3]. The wallet is a compile
time option, `option(ENABLE_WALLET "Enable wallet." ON)`, and can be disabled at
runtime with `-disablewallet` [3]. Core is nonetheless separating the wallet
into its own process, describing the monolithic structure as carrying "increased
security risks due to the tight integration of components" [13].

One consequence is easy to miss: `listunspent` is a wallet RPC, so disabling the
wallet also removes UTXO retrieval. The alternative, `scantxoutset`, is a
brute-force scan of the UTXO set with no incremental mode, which is why
integrators reach for an external indexer [3].

### 1.2 Ethereum relocated it, rather than removing it

Go Ethereum deleted the `personal` namespace in a change merged on 31 October
2024, whose stated aim was that it is "a first step towards removing account
management from geth" [1]. It is worth being precise about what that did and did
not do.

The capability was relocated, not withdrawn. `eth_sendTransaction`, `eth_sign`,
and `eth_signTransaction` all remain defined in the standard execution API
specification [14], so the spec still describes a client backed by a signer.
What geth removed was its own custody of keys.

Geth documents two replacements, and integrators mostly use the second:

| Removed method             | Documented replacement [15]                                |
| -------------------------- | ---------------------------------------------------------- |
| `personal_sendTransaction` | `eth_sendTransaction`, "requiring manual approval in Clef" |
| `personal_signTransaction` | Clef's `account_signTransaction`                           |
| `personal_sign`            | Clef's `account_signData`                                  |
| `personal_newAccount`      | Clef's `account_new`                                       |
| `personal_listAccounts`    | `eth.accounts`                                             |
| `personal_unlockAccount`   | none, by design                                            |

Clef is an external signer that still speaks RPC, and geth describes it as
software that "decouples key management" from the node [16]. The other path,
which is what most integrators take, is a wallet library that holds the key,
signs locally, and submits the signed bytes through the standard namespace.

### 1.3 The others never had it

Solana, Cosmos, Stellar, NEAR, and Sui expose no key-holding method on their
node APIs at all. Two cases are worth stating precisely because their surfaces
invite the wrong reading.

XRPL did not remove signing: `sign` and `wallet_propose` moved from the public
API to admin only in version 1.1.0 [6]. The documentation gives the reasoning
directly: "Unless you run the xrpld server yourself, you should do local signing
using a client library instead of using this command" [6].

Sui's `unsafe_` method prefix denotes unverified inputs, not custody. All
thirteen such methods create an unsigned transaction, and the `signer` parameter
is an address rather than a key [12]. No documentation defines the prefix, so
its intent is `[NOT FOUND]`.

## 2. What the library consumes from the node

The relocation works because the node keeps the transport half of the job. An
Ethereum wallet library assembles a transaction from four standard methods and
submits the result through a fifth:

| What the library needs | Method [14]                                         |
| ---------------------- | --------------------------------------------------- |
| Replay protection      | `eth_chainId`                                       |
| Ordering value         | `eth_getTransactionCount`                           |
| Fee inputs             | `eth_estimateGas`, `eth_gasPrice`, `eth_feeHistory` |
| Submission             | `eth_sendRawTransaction`                            |

The specification names the pattern itself, noting that a caller can "create and
sign a transaction externally using a library such as web3.js or ethers.js"
[14].

This generalises. All eight surveyed chains accept signed bytes on a submission
endpoint, so none forces node-side signing: `eth_sendRawTransaction`,
`sendrawtransaction`, Solana's `sendTransaction`, XRPL's `submit` with a
`tx_blob`, Cosmos `BroadcastTx` over raw `tx_bytes`, Stellar's transaction POST,
NEAR's `send_tx` with `signed_tx_base64`, and Sui's
`sui_executeTransactionBlock` taking transaction bytes plus signatures. Sui's
migration guide states the shape plainly: build with an SDK, "then submit the
resulting bytes" [17].

## 3. Key derivation

| Chain    | Curve                | Standards                          | Coin type                 |
| -------- | -------------------- | ---------------------------------- | ------------------------- |
| Ethereum | secp256k1            | BIP-32, BIP-39, BIP-44             | 60 [2]                    |
| Bitcoin  | secp256k1            | BIP-32, BIP-39, BIP-44             | 0 [4]                     |
| Solana   | ed25519              | SLIP-0010, hardened only           | 501 [5]                   |
| XRPL     | secp256k1 or ed25519 | family seed, and BIP-44 in xrpl.js | 144 [7]                   |
| Cosmos   | secp256k1            | BIP-32, BIP-44, bech32 prefixes    | 118, varies per chain [9] |
| Stellar  | ed25519              | SEP-0005, SLIP-0010                | 148 [10]                  |
| NEAR     | ed25519              | SLIP-0010                          | 397 [11]                  |
| Sui      | ed25519 by default   | SLIP-0010                          | 784 [12]                  |

Cosmos is the outlier on coin type: 118 is the default but chains diverge, with
Terra at 330, Osmosis at 10000118, and Injective at 22000119, while Evmos does
not appear in the SLIP-0044 registry at all [9]. An integrator cannot assume one
path across the Cosmos ecosystem.

### 3.1 Watch-only derivation

The capability an exchange needs most is deriving deposit addresses without
holding the private key. Support splits on the signature curve rather than on
design preference.

| Chain    | Watch-only    | Mechanism or obstacle                                                                              |
| -------- | ------------- | -------------------------------------------------------------------------------------------------- |
| Ethereum | yes           | `neuter()` returns a node that "has no private key, but can be used to derive" child addresses [2] |
| Bitcoin  | yes           | `disable_private_keys` wallets with `importdescriptors` [3]                                        |
| Cosmos   | yes           | secp256k1 public derivation from an extended public key [9]                                        |
| Solana   | no            | "all derivations are hardened" [5]                                                                 |
| Stellar  | no            | SEP-0005 gives two reasons [10]                                                                    |
| NEAR     | no            | named accounts are not key-derived [11]                                                            |
| Sui      | no by default | ed25519 hardened; no first-party extended-public-key interface [12]                                |

The cause is cryptographic. BIP-32 public derivation works on secp256k1 because
a child public key can be computed from a parent public key. SLIP-0010 ed25519
permits only hardened derivation, which requires the private key. Stellar's
SEP-0005 states both halves of its case: ed25519 derivation "does not allow
child key derivation from non-hardened keys", and "each account in Stellar
network is required to hold a minimum balance" [10].

Chains without watch-only derivation adopt a different deposit model. Stellar
uses muxed accounts, which "share a single Stellar account ID across many users,
relying on the memo ID to disambiguate incoming payments" [10]. XRPL uses one
account with destination tags, for the same reserve reason [7].

## 4. Hardware and offline signing

| Chain    | Offline path                             | Hardware                                                          |
| -------- | ---------------------------------------- | ----------------------------------------------------------------- |
| Bitcoin  | PSBT, BIP-174                            | broad support                                                     |
| Ethereum | Clef as external signer [16]             | Ledger, Trezor; HSM via cloud key services [2]                    |
| Solana   | `--sign-only` with an explicit blockhash | "The Solana CLI has first class support for hardware wallets" [5] |
| Cosmos   | Amino JSON only                          | Ledger [9]                                                        |
| Sui      | documented three-step signing path       | TEE and HSM pipeline [12]                                         |
| NEAR     | constrained, see below                   | [NOT FOUND]                                                       |

BIP-174 states the air-gapped case as its purpose: with a partially signed
transaction, "the signer can be offline as all necessary information will be
provided in the transaction" [18].

Two constraints are worth recording. Cosmos hardware signing accepts only
`SIGN_MODE_LEGACY_AMINO_JSON`, and the keyring returns an invalid-sign-mode
error for anything else [9], so the deprecated encoding remains the only
hardware path. And NEAR transactions embed a recent block hash, so a signer must
read node state before signing and the signed result can expire \[11\]: signing
is detachable but not indefinitely offline.

## 5. One core, many languages

Bitcoin Dev Kit is the clearest example of a wallet library built once in Rust
and exported to other languages. The `bdk-ffi` repository "creates a library
ready for export to other languages using uniffi-rs for the Rust-based
bdk_wallet library" [19], shipping Kotlin, Android, and Swift in-repo, with
Kotlin JVM, Python, Dart, and React Native maintained as downstream repositories
consuming the same binding layer [19]. One core reaches six language targets,
and it comes from a community project rather than a foundation.

The nearest equivalent elsewhere is Trust Wallet's `wallet-core`, a C++ core
with Swift and Kotlin bindings covering more than 130 chains [20], likewise
community rather than first-party to any chain. Sui publishes a dedicated
first-party signing library, `rust-signers` [12]. Bitcoin's JavaScript stack
performed the same decomposition internally, splitting key management out of
`bitcoinjs-lib` into the separate `ecpair` and `bip32` packages [4].

## References

01. Go Ethereum, "all: remove `personal` RPC namespace", PR 30704, merged
    2024-10-31. https://github.com/ethereum/go-ethereum/pull/30704
02. ethers.js repository and documentation.
    https://github.com/ethers-io/ethers.js
03. Bitcoin, "Original Bitcoin client RPC API reference".
    https://developer.bitcoin.org/reference/rpc/
04. Bitcoin, "Bitcoin Development" resources page.
    https://bitcoin.org/en/development
05. Solana, "Solana Clients" documentation. https://solana.com/docs/clients
06. XRPL, "sign" method reference, admin API.
    https://xrpl.org/docs/references/http-websocket-apis/admin-api-methods/signing-methods/sign
07. XRPL, "Client Libraries" reference.
    https://xrpl.org/docs/references/client-libraries
08. Cosmos SDK, "CHANGELOG".
    https://raw.githubusercontent.com/cosmos/cosmos-sdk/main/CHANGELOG.md
09. Cosmos, "CosmJS" repository and Cosmos SDK keyring source.
    https://github.com/cosmos/cosmjs
10. Stellar, "SEP-0005: Key Derivation Methods for Stellar Accounts".
    https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0005.md
11. NEAR, "NEAR API" documentation. https://docs.near.org/tools/near-api
12. Sui, "Sui SDKs" reference. https://docs.sui.io/references/sui-sdks
13. Bitcoin Core, "doc/design/multiprocess.md".
    https://raw.githubusercontent.com/bitcoin/bitcoin/master/doc/design/multiprocess.md
14. Ethereum, "execution-apis" specification repository.
    https://github.com/ethereum/execution-apis
15. Go Ethereum, "Personal Namespace" deprecation documentation.
    https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-personal
16. Go Ethereum, "Introduction to Clef".
    https://geth.ethereum.org/docs/tools/clef/introduction
17. Sui, "JSON-RPC Migration" documentation.
    https://docs.sui.io/develop/accessing-data/json-rpc-migration
18. Bitcoin, "BIP-174: Partially Signed Bitcoin Transaction Format".
    https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki
19. Bitcoin Dev Kit, "bdk-ffi" repository.
    https://github.com/bitcoindevkit/bdk-ffi
20. Trust Wallet, "wallet-core" repository.
    https://github.com/trustwallet/wallet-core
