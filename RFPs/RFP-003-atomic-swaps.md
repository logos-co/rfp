---
id: RFP-003
title: Atomic Swaps with LEZ
tier: XL
funding: $TBD
status: draft
category: Applications & Integrations
---

# RFP-003 — Atomic Swaps with LEZ

> **Draft — not yet open for proposals.**
> This RFP will be published once **LEZ timelock support** is available.

## 🧭 Overview

Build a unified atomic swap application that enables trustless, non-custodial exchanges
between LEZ and three major chains: **Bitcoin**, **Monero**, and **Ethereum**. The LEZ
side is implemented as a Risc0 guest program that locks funds contingent on the appropriate cryptographic proof for each chain. A reference implementation for ETH–LEZ swaps already exists ([eth-lez-atomic-swaps](https://github.com/logos-blockchain/eth-lez-atomic-swaps));
this RFP extends the work to Bitcoin and Monero, and delivers a complete, production-ready
swap application.

The application uses a **maker/taker model**: the maker acts as a liquidity provider,
advertising offers over **Logos Delivery** and coordinating swaps over **Logos Chat** —
no central infrastructure required. Teams with experience in applied cryptography (adaptor
signatures, DLEQ proofs), cross-chain protocol design, and Rust/Risc0 development are
best positioned to succeed.


## 🔥 Why This Matters

Trustless swaps between LEZ and widely-held digital assets — without bridges, custodians,
or wrapped tokens — are a prerequisite for meaningful DeFi liquidity in the Logos ecosystem.
Without this primitive, users cannot move value in or out of LEZ without trusting an
intermediary, which directly conflicts with the ecosystem's trust-minimisation principles.

Each target chain presents distinct cryptographic challenges that make this non-trivial.
Bitcoin HTLCs are identifiable on-chain and link the two swap legs; adaptor signatures
with Taproot key-path spends solve this, making swaps indistinguishable from normal
payments. Monero has no scripting system at all, requiring cross-curve DLEQ proofs to
achieve atomicity. Ethereum provides the most flexibility and an existing reference, but
the implementation must integrate with the Logos Ethereum module and meet the same
trustlessness guarantees as the other chains.

Delivering this application — with its fully decentralised coordination via Logos Delivery
and Logos Chat — demonstrates that the Logos stack can support real-world, multi-chain
financial applications without any centralised infrastructure. This is a forcing function
for the maturity of those modules and a high-visibility proof point for ecosystem adoption.


## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. The application must not depend on any centralised server or service. All maker
   advertisement and maker-taker coordination uses Logos Delivery and Logos Chat
   respectively.
2. Trustless swaps between LEZ and **Bitcoin** are supported using Schnorr adaptor
   signatures (BIP-340) and Taproot key-path spends (BIP-341). No custom Bitcoin
   scripts are used; swap transactions are indistinguishable from normal Taproot
   payments.
3. Trustless swaps between LEZ and **Monero** are supported using Ed25519 adaptor
   signatures with cross-curve Discrete Log Equality (DLEQ) proofs (the h4sh3d/COMIT
   protocol). Atomicity is achieved by transferring a Monero spend key share, without
   on-chain scripting.
4. Trustless swaps between LEZ and **Ethereum** are supported using HTLCs or adaptor
   signatures. Ethereum interactions must use the **Logos Ethereum module**.
5. The LEZ escrow program (Rust, Risc0) locks funds contingent on the appropriate
   cryptographic proof for each chain (adaptor secret, DLEQ proof, or hash preimage)
   and releases them upon valid proof submission. Funds are refunded to the depositor
   after the timelock expires.
6. The two legs of each swap are atomic: either both complete or both refund. No
   state exists where one party receives funds and the other does not.
7. Swaps on LEZ support both the native LEZ token and custom tokens issued via the
   LEZ token program, using Associated Token Accounts (ATAs).
8. Swaps on Ethereum must support both **native ETH** and **ERC-20 tokens**. Both
   asset types must be fully supported in the Ethereum escrow contract — partial
   support (e.g., ETH only) is not acceptable.
9. The maker software supports two pricing modes: (1) **local configuration** — static
   prices set via config file or CLI, suitable for testing; (2) **external price feed**
   — prices fetched from an external source (e.g., a REST API). The architecture must
   support pluggable price sources; the specific external integration is left to the
   developer.
10. The maker is deployable as a **headless daemon** covering pair and price
    configuration, external price feed integration, liquidity advertisement, swap
    execution, and monitoring — the daemon must be fully operable via the maker CLI
    without a GUI.

#### Usability

1. Provide a dedicated SDK per trading pair (LEZ–BTC, LEZ–XMR, LEZ–ETH) that can
   be used to build Logos modules for interacting with that pair's swap protocol.
   Each SDK must expose the full swap lifecycle (offer discovery, negotiation,
   escrow creation, claim, and refund) for its respective chain.
2. Provide a **maker daemon** — a long-running headless process that manages
   liquidity advertisement, price feeds, incoming swap requests, and swap execution
   without human interaction. A systemd unit file must be provided for running the
   daemon as a system service, including documented installation steps.
3. Provide a **maker CLI** for operator control of the daemon: configuring trading
   pairs and prices, starting/stopping the daemon, querying swap history, and
   manually triggering claims or refunds. The CLI communicates with the running
   daemon process (e.g., via IPC or a local RPC interface).
4. Provide a **taker CLI** covering the full taker swap lifecycle: discovering maker
   offers, initiating a swap, monitoring progress, and triggering claim or refund.
5. Provide a **maker Logos mini-app GUI** for operator use: configuring pairs and
   prices, monitoring active swaps, and viewing swap history. Loadable in Logos app
   (Basecamp) via git repo with local build instructions and downloadable assets.
6. Provide a **taker Logos mini-app GUI** as the primary taker interface: browsing
   maker offers, initiating swaps, and monitoring progress. Loadable in Logos app
   (Basecamp) via git repo with local build instructions and downloadable assets.
7. Provide an IDL for the LEZ escrow program(s) using the
   [SPEL framework](https://github.com/logos-co/spel).
8. Provide step-by-step documentation for setting up a **Bitcoin Core node**
   (`bitcoind`) on testnet, including configuration, wallet creation, and obtaining
   testnet funds. Documentation must cover both self-hosted and public testnet node
   options.
9. Provide step-by-step documentation for setting up a **Monero node** (`monerod`)
   on stagenet, including wallet RPC configuration (`monero-wallet-rpc`) and
   obtaining stagenet funds. Documentation must cover both self-hosted and public
   stagenet node options.
10. Provide step-by-step documentation for configuring an **Ethereum Web3 RPC
    provider** for Sepolia testnet. All Ethereum interactions in the application must
    use the existing [Logos Ethereum module](https://github.com/logos-blockchain/logos-execution-zone/);
    the documentation must explain how to point the module at a chosen RPC endpoint
    (self-hosted or third-party provider).

#### Reliability

1. **Taker-first on-chain ordering**: The taker must perform their on-chain action
   before the maker performs theirs. The protocol must enforce this ordering — the
   maker must not lock funds on their side until the taker's on-chain transaction is
   confirmed. This ensures the maker is never exposed to loss from a non-responsive
   taker.
2. **On-chain-only execution after lock**: Once the first on-chain action of a swap
   has been submitted, the swap protocol must proceed purely on the basis of on-chain
   state. The application must not depend on Logos Delivery, Logos Chat, or any other
   off-chain channel to complete, claim, or refund a swap in progress. Either party
   must be able to drive the swap to completion (or reclaim funds via timelock) using
   only their local state and the relevant chain nodes.
3. **Graceful degradation**: If a chain-specific dependency is unavailable (e.g., no
   Monero node configured, no Ethereum RPC reachable), the application must still
   start and enable swaps for the remaining chains. Unavailable chains are clearly
   reported to the user.
4. **Swap state persistence**: The swap coordinator must persist swap state locally so
   that incomplete swaps can be resumed after a crash or restart. Loss of swap state
   before the timelock expires must not lead to loss of funds.
5. **Concurrent swap isolation**: Multiple in-flight swaps must not interfere with
   each other. Each swap must maintain independent state, escrow, and timelock tracking.
6. Timelock parameters must account for block time variance, network congestion, and
   clock drift on each chain. The implementation must document timelock parameter
   choices and the rationale.
7. When Logos Delivery or Logos Chat is temporarily unreachable, the application must
   handle this gracefully (e.g., retry, buffering, degraded mode) and document the
   expected behaviour.

#### Performance

1. Document the compute unit usage for each LEZ program operation (initialise escrow,
   claim, refund). LEZ's per-transaction compute budget may change during testnet; the
   submission must note the testnet version against which usage was measured.

#### Supportability

1. The LEZ escrow program is deployed and tested on LEZ testnet 0.2.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode) and are
   included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability, and Performance
   has at least one corresponding test.
5. A reference integration is delivered for each chain demonstrating a complete
   end-to-end swap.
6. A README documents: deployment steps, program addresses, prerequisites for each
   chain (node URLs, wallet requirements, testnet access), and step-by-step setup and
   usage instructions for both maker and taker via CLI and mini-app.
7. The write-up covers: protocol design for each chain, LEZ escrow design,
   cross-chain atomicity argument, timelock handling, security assumptions, and known
   limitations.
8. Each per-pair SDK (LEZ–BTC, LEZ–XMR, LEZ–ETH) must include full API
   documentation: all public types, functions, and error types, with usage examples
   covering the complete swap lifecycle (offer discovery, negotiation, escrow
   creation, claim, and refund).

#### + (Demos)

1. For **each supported chain** (Bitcoin, Monero, Ethereum), three recorded demo
   videos must be submitted:
   - **Happy path**: both parties complete the swap successfully end-to-end.
   - **Refund/timeout path**: one party abandons the protocol and the other recovers
     their funds via the timelock refund.
   - **Concurrent swaps**: two or more swaps executing in parallel, demonstrating
     correct handling of multiple in-flight swaps.

### Soft Requirements

- Support for the **Logos Ethereum module** could be extended to additional EVM-compatible chains in a single submission.


## 👤 Recommended Team Profile

- Applied cryptography: adaptor signatures (Schnorr/secp256k1), Ed25519, cross-curve
  DLEQ proofs
- Cross-chain protocol design and atomicity guarantees
- Rust development and the Risc0 zkVM toolchain
- Bitcoin: Taproot/P2TR transaction construction, BIP-340/BIP-341
- Monero: key structure (spend/view keys), transaction construction, stagenet
- Ethereum: Solidity smart contracts, Foundry or Hardhat, ERC-20 interactions
- Distributed systems: swap state machines, crash recovery, concurrent coordination


## ⏱ Timeline Expectations

Estimated duration: **16–24 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

### General

- [eth-lez-atomic-swaps](https://github.com/logos-blockchain/eth-lez-atomic-swaps) — ETH–LEZ HTLC-based swap (reference implementation and LEZ program structure)
- [Logos Execution Zone](https://github.com/logos-blockchain/logos-execution-zone/)
- [Risc0 proving system](https://dev.risczero.com/)

### Bitcoin

- [BIP-340: Schnorr signatures for secp256k1](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [BIP-341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)
- [Adaptor signatures — Lloyd Fournier](https://github.com/LLFourn/one-time-vrf/blob/master/main.pdf)
- [Scriptless Scripts — Andrew Poelstra](https://github.com/apoelstra/scriptless-scripts)
- [rust-bitcoin](https://github.com/rust-bitcoin/rust-bitcoin)
- [secp256k1-zkp (adaptor signature support)](https://github.com/BlockstreamResearch/secp256k1-zkp)

### Monero

- [Bitcoin–Monero Cross-chain Atomic Swap — h4sh3d paper](https://eprint.iacr.org/2020/1126.pdf)
- [comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap) — production Monero-Bitcoin implementation
- [comit-network/cross-curve-dleq](https://github.com/comit-network/cross-curve-dleq) — cross-group DLEQ proof library (secp256k1 ↔ Ed25519)
- [curve25519-dalek](https://github.com/dalek-cryptography/curve25519-dalek)
- [secp256kFUN!](https://github.com/LLFourn/secp256kfun)

### Ethereum

- [Logos Ethereum module](https://github.com/logos-blockchain/logos-execution-zone/) — use for all Ethereum interactions


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use **Discussions**.
