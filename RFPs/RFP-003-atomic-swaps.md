---
id: RFP-003
title: Atomic Swaps with LEZ
tier: XL
status: closed
category: Applications & Integrations
dependencies:
  - id: LEZ-timelock
    reason: Platform feature. The escrow program refunds the depositor after the timelock expires (F5). Delivered via the LEZ clock program, whose timestamp accounts let a program enforce deadlines.
---

# RFP-003 — Atomic Swaps with LEZ

> **Status:** Closed. This RFP is no longer accepting proposal submissions.

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are the
> criteria against which proposals and milestones are evaluated. Logos makes no
> representation as to the legal or regulatory treatment of this specification or
> any implementation of it in any jurisdiction.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) for complying with any legal and regulatory requirements that apply
> to them. Software developed under the Program is published and maintained by
> its developers, not by Logos.
>
> Anyone who chooses to deploy, host, operate or use software developed under the
> Program, whether or not they were awarded a grant under the Program, does so at
> their own risk and is solely responsible for complying with any legal or
> regulatory requirements that apply to them. See the
> [Terms & Conditions](../TERMS_AND_CONDITIONS.md).
>
> Deploying the software described in this RFP, operating any service based on
> it, or carrying on business through it may amount to regulated activity in some
> jurisdictions, including where it involves holding or managing users' assets or
> providing services to others. Whoever conducts any such activity does so as
> principal, in their own name, and is solely responsible for assessing its
> regulatory treatment, including any licensing, registration, sanctions or
> anti-money laundering obligations that may apply to them. Logos does not make
> any representation, provides any advice or assumes any responsibility in
> respect of any such determination or compliance.

## 🧭 Overview

Build a unified atomic swap application that enables trustless, non-custodial
exchanges between LEZ and three external chains: **Bitcoin**, **Monero**, and
**Zcash** (transparent pool). The LEZ side is implemented as a Risc0 guest
program that locks funds contingent on the appropriate cryptographic proof for
each chain. An HTLC-based reference implementation for ETH and LEZ already
exists
([eth-lez-atomic-swaps](https://github.com/logos-co/eth-lez-atomic-swaps)) and
demonstrates the LEZ escrow structure; this RFP applies that foundation to
Bitcoin, Monero, and Zcash, and delivers a complete, production-ready swap
application. An Ethereum pair is out of scope: ETH is expected to reach LEZ via
wrapping, which requires no swap counterparty and is a much simpler
construction. Zcash shielded-pool (Sapling / Orchard) swaps are explicitly out
of scope: the cryptographic state of the art does not yet provide a drop-in
design (see
[Appendix: Zcash Atomic Swap Primitives](../appendix/zcash-atomic-swap-primitives.md)).

The application uses a **maker/taker model**: the maker acts as a liquidity
provider, advertising offers over **Logos Delivery** and coordinating swaps over
**Logos Chat**, with no central infrastructure required. Teams with experience
in applied cryptography (adaptor signatures, DLEQ proofs), cross-chain protocol
design, and Rust/Risc0 development are best positioned to succeed.

## 🔥 Why This Matters

Trustless swaps between LEZ and widely-held digital assets, without bridges,
custodians, or wrapped tokens, are a prerequisite for meaningful DeFi liquidity
in the Logos ecosystem. Without this primitive, users cannot move value in or
out of LEZ without trusting an intermediary, which directly conflicts with the
ecosystem's trust-minimisation principles.

Each target chain presents distinct cryptographic challenges that make this
non-trivial. Bitcoin HTLCs use a distinctive script pattern and place the same
hash value on both swap legs; adaptor signatures with Taproot key-path spends
avoid both, so cooperative swap transactions take the standard form of ordinary
Taproot spends. Monero has no scripting system at all, requiring cross-curve
DLEQ proofs to achieve atomicity. Zcash transparent inherits Bitcoin script
unchanged and supports HTLC-based swaps via
[BIP-199](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki);
adaptor-signature variants are a stretch goal for removing the shared hash
identifier that HTLCs place on both chains. The
Bitcoin and Monero constructions are surveyed with sources in
[Appendix: Bitcoin and Monero Adaptor-Signature Swap Primitives](../appendix/btc-xmr-adaptor-swap-primitives.md);
the Zcash analysis is in
[Appendix: Zcash Atomic Swap Primitives](../appendix/zcash-atomic-swap-primitives.md).

Delivering this application, with its fully decentralised coordination via Logos
Delivery and Logos Chat, shows that the Logos stack can support multi-chain
financial applications without any centralised infrastructure. It exercises
those modules under a real workload and gives the ecosystem a concrete swap
application to build on.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. The application must not depend on any centralised server or service. All
   maker advertisement and maker-taker coordination uses Logos Delivery and
   Logos Chat respectively.
2. Trustless swaps between LEZ and **Bitcoin** are supported using Schnorr
   adaptor signatures (BIP-340) and Taproot (BIP-341). The cooperative claim
   path must be a key-path spend, in the standard form of an ordinary Taproot
   payment, with no protocol-specific script footprint on-chain. The **refund
   branch** may be implemented either as a pre-signed timelocked transaction
   (key-path, refund in the same standard form) or as a Taproot script-path
   tapleaf (`OP_CHECKSEQUENCEVERIFY`, refund spend
   identifiable but enforced by consensus); both are acceptable. The two
   constructions and their trade-offs are described in
   [Appendix: Bitcoin and Monero Adaptor-Signature Swap Primitives](../appendix/btc-xmr-adaptor-swap-primitives.md).
   Absent a measured reason to prefer otherwise (see Reliability), the simpler
   script-path refund is the recommended default.
3. Trustless swaps between LEZ and **Monero** are supported using Ed25519
   adaptor signatures with cross-curve Discrete Log Equality (DLEQ) proofs (the
   h4sh3d/COMIT protocol). Atomicity is achieved by transferring a Monero spend
   key share, without on-chain scripting.
4. Trustless swaps between LEZ and **Zcash transparent (`t1...`) addresses** are
   supported using BIP-199-style HTLCs
   (`OP_IF / OP_SHA256 / OP_CLTV / OP_CHECKSIG` on the Zcash side, mirrored on
   the LEZ side). The Zcash refund deadline must strictly succeed the LEZ refund
   deadline by a margin documented to cover worst-case Zcash confirmation
   latency. ECDSA adaptor signatures (Lloyd Fournier construction) are an
   acceptable alternative to HTLCs; unlike an HTLC an adaptor-signature
   construction does not place the same hash value on both chains so the two legs
   of a swap do not share a common on-chain identifier and are encouraged for
   cross-chain unlinkability. Zcash shielded-pool (Sapling / Orchard) swaps are
   **out of
   scope** for this RFP; see
   [Appendix: Zcash Atomic Swap Primitives](../appendix/zcash-atomic-swap-primitives.md)
   for the rationale.
5. The LEZ escrow program (Rust, Risc0) locks funds contingent on the
   appropriate cryptographic proof for each chain (adaptor secret, DLEQ proof,
   or hash preimage) and releases them upon valid proof submission. Funds are
   refunded to the depositor after the timelock expires.
6. The two legs of each swap are atomic: either both complete or both refund. No
   state exists where one party receives funds and the other does not.
7. Swaps on LEZ support both the native LEZ token and custom tokens issued via
   the LEZ token program, using Associated Token Accounts (ATAs).
8. The maker software supports two pricing modes: (1) **local configuration**:
   static prices set via config file or CLI, suitable for testing; (2)
   **external price feed**: prices sourced from another Logos module via its C
   API. The architecture must support pluggable price sources; the specific Logos
   module and its C API integration are left to the developer.
9. The maker is deployable as a **headless daemon** covering pair and price
   configuration, external price feed integration, liquidity advertisement, swap
   execution, and monitoring; the daemon must be fully operable via the maker
   CLI without a GUI.

#### Usability

01. Provide a dedicated SDK per trading pair (LEZ–BTC, LEZ–XMR, LEZ–ZEC) that
    can be used to build Logos modules for interacting with that pair's swap
    protocol. Each SDK must expose the full swap lifecycle (offer discovery,
    negotiation, escrow creation, claim, and refund) for its respective chain.
02. Provide a **maker daemon**: a long-running headless process that manages
    liquidity advertisement, price feeds, incoming swap requests, and swap
    execution without human interaction. A systemd unit file must be provided
    for running the daemon as a system service, including documented
    installation steps.
03. Provide a **maker CLI** for operator control of the daemon: configuring
    trading pairs and prices, starting/stopping the daemon, querying swap
    history, and manually triggering claims or refunds. The CLI communicates
    with the running daemon process (e.g., via IPC or a local RPC interface).
04. Provide a **taker CLI** covering the full taker swap lifecycle: discovering
    maker offers, initiating a swap, monitoring progress, and triggering claim
    or refund.
05. Provide a **maker Logos mini-app GUI** for operator use: configuring pairs
    and prices, monitoring active swaps, and viewing swap history. Loadable in
    Logos app (Basecamp) via git repo with local build instructions and
    downloadable assets.
06. Provide a **taker Logos mini-app GUI** as the primary taker interface:
    browsing maker offers, initiating swaps, and monitoring progress. Loadable
    in Logos app (Basecamp) via git repo with local build instructions and
    downloadable assets.
07. Provide an IDL for the LEZ escrow program(s) using the
    [SPEL framework](https://github.com/logos-co/spel).
08. Provide step-by-step documentation for setting up a **Bitcoin Core node**
    (`bitcoind`) on testnet, including configuration, wallet creation, and
    obtaining testnet funds. Documentation must cover both self-hosted and
    public testnet node options.
09. Provide step-by-step documentation for setting up a **Monero node**
    (`monerod`) on stagenet, including wallet RPC configuration
    (`monero-wallet-rpc`) and obtaining stagenet funds. Documentation must cover
    both self-hosted and public stagenet node options.
10. Provide step-by-step documentation for setting up a **Zcash node** (`zcashd`
    or `zebrad`) on testnet, including transparent wallet creation and obtaining
    testnet funds. Documentation must cover both self-hosted and public testnet
    node options.

#### Reliability

1. **Taker-first on-chain ordering**: The taker must perform their on-chain
   action before the maker performs theirs. The protocol must enforce this
   ordering: the maker must not lock funds on their side until the taker's
   on-chain transaction is confirmed. This ensures the maker is never exposed to
   loss from a non-responsive taker.
2. **On-chain-only execution after lock**: Once the first on-chain action of a
   swap has been submitted, the swap protocol must proceed purely on the basis
   of on-chain state. The application must not depend on Logos Delivery, Logos
   Chat, or any other off-chain channel to complete, claim, or refund a swap in
   progress. Either party must be able to drive the swap to completion (or
   reclaim funds via timelock) using only their local state and the relevant
   chain nodes.
3. **Graceful degradation**: If a chain-specific dependency is unavailable
   (e.g., no Monero node configured, no Zcash node reachable), the application
   must still start and enable swaps for the remaining chains. Unavailable
   chains are clearly reported to the user.
4. **Swap state persistence**: The swap coordinator must persist swap state
   locally so that incomplete swaps can be resumed after a crash or restart.
   Loss of swap state before the timelock expires must not lead to loss of
   funds.
5. **Concurrent swap isolation**: Multiple in-flight swaps must not interfere
   with each other. Each swap must maintain independent state, escrow, and
   timelock tracking.
6. Timelock parameters must account for block time variance, network congestion,
   and clock drift on each chain. The implementation must document timelock
   parameter choices and the rationale.
7. **Bitcoin refund construction must be justified.** The write-up must state
   which refund branch construction was chosen for the Bitcoin pair (pre-signed
   timelocked key-path transaction, or Taproot script-path tapleaf) and justify
   it against the trade-off in the appendix. If the pre-signed approach is
   chosen for its refund-path privacy, the submission must enumerate the
   additional failure modes it introduces (lost or corrupted refund transaction,
   malformed timelock, refund-tx fee rate fixed at setup becoming
   unbroadcastable, dependence on timelock ordering rather than consensus) and
   describe how each is mitigated, so the privacy benefit can be weighed against
   the measured increase in fragility.
8. When Logos Delivery or Logos Chat is temporarily unreachable, the application
   must handle this gracefully (e.g., retry, buffering, degraded mode) and
   document the expected behaviour.

#### Performance

1. Document the compute unit usage for each LEZ program operation (initialise
   escrow, claim, refund). LEZ's per-transaction compute budget may change
   during testnet; the submission must note the testnet version against which
   usage was measured.

#### Supportability

01. The LEZ escrow program is deployed and tested on LEZ testnet 0.2.
02. End-to-end integration tests run against a LEZ sequencer (standalone mode)
    and are included in CI.
03. CI must be green on the default branch.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
05. A reference integration is delivered for each chain demonstrating a complete
    end-to-end swap.
06. A README documents: deployment steps, program addresses, prerequisites for
    each chain (node URLs, wallet requirements, testnet access), and
    step-by-step setup and usage instructions for both maker and taker via CLI
    and mini-app.
07. The write-up covers: protocol design for each chain, LEZ escrow design,
    cross-chain atomicity argument, timelock handling, security assumptions, and
    known limitations.
08. Each per-pair SDK (LEZ–BTC, LEZ–XMR, LEZ–ZEC) must include full API
    documentation: all public types, functions, and error types, with usage
    examples covering the complete swap lifecycle (offer discovery, negotiation,
    escrow creation, claim, and refund).
09. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for each per-pair SDK (LEZ–BTC, LEZ–XMR, LEZ–ZEC), covering the developer
    integration journey for that swap protocol.
10. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the maker CLI and a separate one for the taker CLI, each covering the
    respective operator/user journey end-to-end.
11. Provide Figma designs or equivalent for the maker mini-app GUI and the taker
    mini-app GUI.
12. A formal review of all smart contracts and on-chain scripts must be obtained
    from a reputable third party. This covers the LEZ escrow program (Rust,
    Risc0) and every external-chain script used by the swaps (Bitcoin Taproot
    scripts and pre-signed transactions, Zcash transparent HTLC scripts, and any
    other on-chain locking logic). The review report and any resulting
    remediations must be included in the submission.
13. A formal review of the cross-chain swap protocol implementation must be
    obtained from a reputable third party, covering the atomicity guarantees,
    taker-first ordering, timelock handling, and per-chain adaptor-signature or
    HTLC constructions. The review report and any resulting remediations must be
    included in the submission.

#### + (Demos)

1. For **each supported chain** (Bitcoin, Monero, Zcash), three recorded demo
   videos must be submitted:
   - **Happy path**: both parties complete the swap successfully end-to-end.
   - **Refund/timeout path**: one party abandons the protocol and the other
     recovers their funds via the timelock refund.
   - **Concurrent swaps**: two or more swaps executing in parallel,
     demonstrating correct handling of multiple in-flight swaps.

### Soft Requirements

- An **adaptor-signature variant of the LEZ–ZEC pair** (replacing BIP-199 HTLCs),
  so that the two legs of a swap do not share a common on-chain identifier for
  cross-chain unlinkability, as described in
  [Appendix: Zcash Atomic Swap Primitives](../appendix/zcash-atomic-swap-primitives.md).

## 👤 Recommended Team Profile

- Applied cryptography: adaptor signatures (Schnorr/secp256k1), Ed25519,
  cross-curve DLEQ proofs
- Cross-chain protocol design and atomicity guarantees
- Rust development and the Risc0 zkVM toolchain
- Bitcoin: Taproot/P2TR transaction construction, BIP-340/BIP-341
- Monero: key structure (spend/view keys), transaction construction, stagenet
- Zcash: transparent transaction construction, BIP-199 HTLCs, `zcashd`/`zebrad`
  testnet
- Distributed systems: swap state machines, crash recovery, concurrent
  coordination

## ⏱ Timeline Expectations

Estimated duration: **16–24 weeks**

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

### General

- [eth-lez-atomic-swaps](https://github.com/logos-co/eth-lez-atomic-swaps):
  ETH–LEZ HTLC-based swap (prior art for the LEZ program structure; an ETH pair
  is itself out of scope)
- [Logos Execution Zone](https://github.com/logos-blockchain/logos-execution-zone/)
- [Risc0 proving system](https://dev.risczero.com/)

### Bitcoin

- [Appendix: Bitcoin and Monero Adaptor-Signature Swap Primitives](../appendix/btc-xmr-adaptor-swap-primitives.md):
  sourced survey of the constructions required by this RFP
- [BIP-340: Schnorr signatures for secp256k1](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [BIP-341: Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)
- [DLC specs, Adaptor Signatures](https://github.com/discreetlogcontracts/dlcspecs/blob/master/AdaptorSignature.md):
  production-grade spec for both Schnorr and ECDSA adaptor signatures with test
  vectors
- [Aumayr et al. (2021), *Generalized Channels from Limited Blockchain Scripts and Adaptor Signatures*](https://eprint.iacr.org/2020/476):
  formal security definitions (aEUF-CMA, witness-extractability, pre-signature
  adaptability)
- [Adaptor signatures, Lloyd Fournier](https://github.com/LLFourn/one-time-VES):
  reference ECDSA adaptor construction with DLEQ proof
- [Scriptless Scripts — Andrew Poelstra](https://github.com/apoelstra/scriptless-scripts)
- [rust-bitcoin](https://github.com/rust-bitcoin/rust-bitcoin)
- [secp256k1-zkp (adaptor signature support)](https://github.com/BlockstreamResearch/secp256k1-zkp)
- [secp256kfun / ecdsa-fun](https://github.com/LLFourn/secp256kfun): Rust
  libraries with clean reference implementations of Schnorr and ECDSA adaptor
  signatures

### Monero

- [Appendix: Bitcoin and Monero Adaptor-Signature Swap Primitives](../appendix/btc-xmr-adaptor-swap-primitives.md):
  sourced survey of the constructions required by this RFP
- [Bitcoin–Monero Cross-chain Atomic Swap — h4sh3d paper](https://eprint.iacr.org/2020/1126.pdf)
- [comit-network/xmr-btc-swap](https://github.com/comit-network/xmr-btc-swap):
  production Monero-Bitcoin implementation
- [comit-network/cross-curve-dleq](https://github.com/comit-network/cross-curve-dleq):
  cross-group DLEQ proof library (secp256k1 ↔ Ed25519)
- [curve25519-dalek](https://github.com/dalek-cryptography/curve25519-dalek)
- [secp256kFUN!](https://github.com/LLFourn/secp256kfun)

### Zcash

- [Appendix: Zcash Atomic Swap Primitives](../appendix/zcash-atomic-swap-primitives.md):
  feasibility matrix, why shielded is out of scope, prior-art comparison
- [BIP-199: Hashed Time-Locked Contract transactions](https://github.com/bitcoin/bips/blob/master/bip-0199.mediawiki):
  canonical HTLC layout used for transparent ZEC swaps
- [ZIP-203: Transaction Expiry](https://zips.z.cash/zip-0203): `nExpiryHeight`
  semantics
- [ZIP-202: Version 3 Transaction Format for Overwinter](https://zips.z.cash/zip-0202)
- [`zcash/zcash` (`zcashd`)](https://github.com/zcash/zcash) and
  [`ZcashFoundation/zebra` (`zebrad`)](https://github.com/ZcashFoundation/zebra):
  full nodes
- [`librustzcash`](https://github.com/zcash/librustzcash): Rust
  transaction-construction crates
- [Zwap forum thread](https://forum.zcashcommunity.com/t/zwap-unlinkable-cross-chain-atomic-swaps/55104):
  most active 2025 prior-art for transparent ZEC atomic swaps

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
