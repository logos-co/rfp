---
id: RFP-021
title: Wrapped ERC-20 Bridge for LEZ
tier: L
status: open
category: Developer Tooling & Infrastructure
dependencies:
  - id: RFP-001
    reason: Admin authority governs the supported-token registry and the attestor federation's membership and threshold, as specified in Functionality.
  - id: RFP-002
    reason: Freeze authority provides the circuit breaker to halt minting and/or redemption, globally or per token, if the attestor federation or the Ethereum-side vault is suspected compromised.
  - id: LP-0013
    reason: Mint/burn token authority primitives are required for the LEZ program to mint wrapped tokens on deposit and burn them on redemption.
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-021 — Wrapped ERC-20 Bridge for LEZ

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
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
> does not make any representation, provides any advice or assumes any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

Build a trustless lock-and-mint bridge that lets ERC-20 tokens (and native ETH, wrapped as
WETH) held on Ethereum enter LEZ as canonical wrapped assets, and exit back to
Ethereum on redemption. An Ethereum-side vault contract escrows deposits; a
RISC0-based cryptographic proof of the deposit (verifying Ethereum consensus and
state) is generated and submitted to a LEZ-side program, which verifies the proof
in-program and mints the corresponding wrapped token to the recipient's LEZ
account. Redemption reverses the flow: burning the wrapped token on LEZ triggers
generation of a zero-knowledge proof of the burn, which is verified on-chain by
the Ethereum vault via a native precompile, releasing the original ERC-20 from
the vault. The bridge core depends on mature RISC0 implementations (e.g. Zisk)
for consensus and state proof generation, requiring teams experienced with
zero-knowledge proof systems, Solidity smart-contract development, LEZ program
development, and RISC0 guest environments.

## 🔥 Why This Matters

LEZ DeFi cannot function without external collateral. The lending protocol
([RFP-008](./RFP-008-lending-borrowing-protocol.md)), the reflexive stablecoin
([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), and the on-chain TWAP
oracle's own design ([RFP-019](./RFP-019-twap-oracle.md)) all assume wrapped
external assets — wBTC, wETH, wXMR, wZEC — are already available as LEZ tokens.
None of those RFPs specify how a token actually gets wrapped; this RFP is that
missing primitive for the Ethereum leg (Bitcoin, Monero, and Zcash have their
own trustless path via atomic swaps in
[RFP-003](./RFP-003-atomic-swaps.md)).

RFP-003 explicitly carved Ethereum out of its scope for exactly this reason:
"ETH is expected to reach LEZ via wrapping, which requires no swap counterparty
and is a much simpler construction." Bitcoin, Monero, and Zcash lack general
smart-contract expressiveness, so a trustless swap protocol (adaptor
signatures, HTLCs, DLEQ proofs) is the only construction available for them.
Ethereum's programmability makes a lock-and-mint bridge with a vault contract
and zero-knowledge proof verification possible instead, which is the trustless
architecture used to move value onto smart-contract chains that verify
cryptographic proofs natively. This RFP delivers that trustless construction
via RISC0 zero-knowledge proofs, matching LEZ's cryptographic verification
capability with Ethereum's native precompile support for proof verification.

Stablecoins are the concrete prize. USDT and USDC together account for over 80%
of a stablecoin market that stood above $300B in mid-2026, and are the
settlement asset most DeFi money markets and DEXes actually run on. Without a
wrapped ERC-20 primitive, LEZ has no path to bring USDC, USDT, DAI, or WETH
liquidity onto the chain, which blocks the lending protocol and the reflexive
stablecoin from having any credible collateral base at launch.

Bridges are also the most attacked category of infrastructure in DeFi:
Chainalysis has tracked more than $2.8B stolen from cross-chain bridges since
2022, the highest-value class of exploit in the industry. This RFP's security
posture — cryptographic verification via RISC0 zero-knowledge proofs, eliminating
the trust-in-signers vector entirely, combined with per-token and global
deposit/redemption caps and an admin-governed freeze-authority circuit
breaker — is designed directly against that track record. The trustless
cryptographic core ensures that no signer compromise, key theft, or participant
collusion can lead to spurious minting or release, and is explained in full in
Design Rationale and the Bridge Security requirements below.

## 🏗 Design Rationale

### Trustless lock-and-mint via zero-knowledge proofs

The bridge has two independently deployed halves unified by a cryptographic
proof-verification primitive:

1. **Ethereum vault contract.** A Solidity contract that accepts deposits of any
   ERC-20 token in the supported-token registry (plus native ETH, which the
   vault wraps to WETH internally so ETH does not need a separate bridging
   path). Each deposit locks the token and emits a
   `Deposit(token, depositor, amount, lezRecipient, nonce)` event.
2. **RISC0 proof generator (inbound).** An off-chain service (leveraging mature
   RISC0 implementations such as Zisk) that observes Ethereum deposits reaching
   finality, generates a zero-knowledge proof verifying (1) the Ethereum
   consensus state at a specific block height, and (2) the deposit event state
   in that block. The proof is deterministically derived from the deposit tuple
   `(token, depositor, amount, lezRecipient, nonce)` and Ethereum chain state.
3. **LEZ bridge program (inbound).** A public-mode LEZ program verifies the RISC0
   proof in-program (validating Ethereum consensus and the inclusion of the
   deposit event) and mints the corresponding wrapped token to `lezRecipient`,
   crediting a private account directly where the recipient requests it (see
   "+ Privacy" below). The deposit `nonce` is recorded on first use and any
   repeat submission of the same nonce is rejected, preventing double-mint.

No signer set, threshold signature, or attestor federation is required. The
proof's correctness is cryptographically ensured by the RISC0 circuit; no trust
in external parties is introduced.

### Redemption: burn-and-release

Redemption is the mirror flow. A holder burns wrapped tokens on LEZ via the
bridge program, which validates the burn and emits a structured redemption
event (per [LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md))
carrying `(token, amount, ethereumRecipient, redemptionNonce)`. Once the burn
transaction reaches LEZ finality, an off-chain service generates a RISC0
zero-knowledge proof verifying (1) the LEZ consensus state at a specific epoch,
and (2) the burn event in that state. The Ethereum vault receives the proof and
verifies it natively via a precompile (Groth16 verifier on-chain or equivalent)
— a single verification step, orders of magnitude cheaper than the in-program
RISC-V path and matching the cost asymmetry documented in RFP-020 between
LEZ-side and EVM-side cryptographic verification. Upon successful proof
verification, the vault releases the original ERC-20 to `ethereumRecipient`,
tracking `redemptionNonce` to reject replays on the Ethereum side.

### Token registry and decimal normalisation

Each supported ERC-20 is registered individually by the admin authority: its
Ethereum contract address, its LEZ wrapped-token mint, its decimals, and its
per-token deposit/redemption cap (see "Bridge Security" below). ERC-20 tokens
do not share a common decimals convention (6 for USDC, 8 for WBTC, 18 for WETH
and DAI); the wrapped LEZ mint for each token must document its own decimals
and the exact conversion applied on mint and burn, and the registration
instruction must reject a token whose decimals cannot be represented exactly in
the chosen LEZ mint precision.

### Finality windows and reorg protection

An attestor must not sign a deposit attestation until the deposit transaction
has reached a configured Ethereum finality depth (a block-count or, once
generally available, a beacon-chain finalised-checkpoint condition). Signing
before finality risks attesting to a deposit that a reorg later removes, which
would mint wrapped tokens against a deposit that no longer exists on Ethereum.
The chosen depth is a direct trade-off between user-facing deposit latency and
reorg risk, and proposals must document the depth chosen and the residual risk
it leaves. The reverse direction (LEZ burn to Ethereum release) is symmetric:
attestors wait for LEZ finality on the burn transaction before co-signing a
release.

### Trust model: cryptographic, trustless

This bridge is **trustless**. It requires no trust in external signers, validators,
or attestor federations. A user who bridges an ERC-20 asset relies only on the
correctness of the RISC0 circuit (verifying Ethereum consensus and state inclusion),
the security of the Ethereum network itself, and the security of the LEZ network.
Both inbound and outbound flows are verified by immutable cryptographic proofs,
not by signatures from a bounded set of participants.

Proposals must state this trustlessness explicitly in user-facing documentation
(mini-app, README, SDK docs). Per-token and global deposit/redemption caps, and
the freeze-authority circuit breaker (see Bridge Security), remain in place as
operational safety mechanisms independent of the cryptographic trust model. They
bound the operational blast radius and provide an administrative pause capability,
but are orthogonal to the bridge's trustlessness.

### Fee structure

This RFP does not mandate a specific fee rate. Proposals must specify who pays
(depositor, redeemer, or both), when fees are collected (deposit, redemption,
or both), the exact rate, and where fees are routed. A governance-activatable
fee switch with an initial zero rate, gated by the admin authority per
RFP-001, is the recommended baseline, consistent with the pattern used
elsewhere in the Logos RFP set (see
[RFP-017](./RFP-017-token-vesting.md), "Fee structure").

## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

1. Implement an Ethereum vault contract (Solidity) that accepts deposits of any
   ERC-20 token in the supported-token registry, plus native ETH (auto-wrapped
   to WETH by the vault). Each deposit locks the token and emits a
   `Deposit(token, depositor, amount, lezRecipient, nonce)` event with a
   monotonically increasing `nonce`.
2. Implement an inbound RISC0 proof generator that observes finalised Ethereum
   deposits, constructs a proof of the deposit (verifying Ethereum consensus at
   a configured finality depth and the deposit event's inclusion in that state),
   and outputs a proof artifact ready for on-chain verification. The proof must
   encode `(token, depositor, amount, lezRecipient, nonce)` and be
   deterministically reproducible from chain state.
3. Implement a public-mode LEZ bridge program that accepts an inbound RISC0
   proof, verifies it in-program (validating Ethereum consensus and state
   inclusion), and mints the corresponding wrapped token to `lezRecipient` — to
   a public or a private LEZ account, at the recipient's choice. Each deposit
   `nonce` may be consumed at most once; a repeat submission of an
   already-processed nonce must be rejected without minting.
4. Implement a burn instruction on the LEZ bridge program: a holder burns a
   specified amount of a wrapped token, and the program emits a structured
   redemption event (per LP-0012) carrying `(token, amount, ethereumRecipient,
   redemptionNonce)`.
5. Implement an outbound RISC0 proof generator that observes finalised LEZ burn
   events, constructs a proof of the burn (verifying LEZ consensus at a
   configured finality depth and the burn event's inclusion in that state), and
   outputs a proof artifact ready for on-chain Ethereum verification. The proof
   must encode `(token, amount, ethereumRecipient, redemptionNonce)` and be
   deterministically reproducible from chain state.
6. The Ethereum vault contract accepts an outbound RISC0 proof and verifies it
   natively via a precompile (Groth16 verifier or equivalent), releasing the
   original ERC-20 (or unwrapped ETH, for WETH redemptions) to
   `ethereumRecipient`. Each `redemptionNonce` may be consumed at most once.
7. An admin authority (per RFP-001, integrated via the SPEL framework where
   applicable to the LEZ side) can: register a new supported ERC-20 (Ethereum
   address, LEZ wrapped mint, decimals, per-token deposit/redemption caps), and
   deregister a token. Registration changes must be mirrored consistently on
   both the Ethereum vault and the LEZ bridge program (document how the two
   sides stay in sync and what happens if they temporarily diverge).
8. A freeze authority (per RFP-002) can pause minting and/or redemption, either
   globally or for a single registered token, on both the Ethereum vault and
   the LEZ bridge program independently.
9. Global and per-token deposit and redemption caps (configurable by the admin
   authority) bound the maximum value that can be minted or released within a
   rolling window, as a rate-limiting circuit breaker independent of the
   freeze authority.

#### Usability

1. Provide an SDK that can be used to build Logos modules for: submitting
   deposits to the Ethereum vault, querying deposit/proof/mint status, querying
   inbound proof generation progress, initiating a redemption (burn), querying
   redemption/proof/release status, and reading the supported-token registry.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. The mini-app must
   cover: a deposit flow (connect an Ethereum wallet, approve and deposit,
   track proof generation progress, confirm LEZ mint, choose public or private
   destination account), a redemption flow (burn wrapped tokens, track proof
   generation, confirm Ethereum release), and a registry view (supported tokens,
   per-token caps and current utilisation).
3. Provide a CLI that covers core functionality: deposit, query status, redeem,
   and query the token registry. The CLI may have fewer features than the
   mini-app but must support all essential operations.
4. Provide the inbound and outbound RISC0 proof generators as **Logos modules
   accompanied by Logos Core headless CLI/daemons**, so operators can run them
   as standalone long-running processes. Each daemon must support: configurable
   Ethereum and/or LEZ RPC endpoints, configurable finality depth for the source
   chain, structured logging of observed events and generated proofs, and a
   clean shutdown path. The proof generators must integrate with or wrap mature
   RISC0 implementations (e.g. Zisk) without reimplementing zero-knowledge
   primitives. Document the operator journey end-to-end: install, configure, run,
   monitor.
5. Provide an IDL for the LEZ bridge program using the
   [SPEL framework](https://github.com/logos-co/spel).
6. Return clear, actionable error messages for all failure modes: unsupported
   token, cap exceeded, invalid proof, proof verification failure, insufficient
   finality, nonce already processed, and program or per-token frozen.

#### Reliability

1. Minting is atomic: a failed or rejected mint attempt does not consume the
   deposit nonce and leaves the deposit mintable on retry.
2. Burn-and-redeem is atomic: a failed redemption attempt does not burn tokens
   without a corresponding, retryable redemption record.
3. A deposit or redemption nonce can be consumed at most once; replaying an
   already-verified proof is rejected deterministically and does not
   double-mint or double-release.
4. Proof generation is deterministic: given the same chain state and input
   parameters, proof generators must produce the same proof (or provably
   equivalent proofs if the RISC0 implementation supports non-determinism).
   This ensures that multiple independent proof generators can verify each
   other's work and that replaying a proof from cache is safe.
5. A temporary RPC or connectivity failure on either chain leaves the proof
   generator daemon in a recoverable state, able to resume observing and
   generating proofs once connectivity is restored, without needing to replay
   already-proven items.

#### Performance

1. Verifying an inbound RISC0 proof (Ethereum consensus + state inclusion,
   in-program on LEZ) and minting must complete within a single LEZ public
   transaction at the per-transaction compute budget in force at delivery time.
   Document the compute-unit cost, including the cost breakdown for the RISC0
   proof verification and the LEZ mint instruction, extending the measurement
   methodology from [RFP-020](./RFP-020-redstone-oracle-adaptor.md).
2. Document the Ethereum-side gas cost of verifying the outbound RISC0 proof
   via a precompile (Groth16 verifier or equivalent) and releasing funds from
   the vault. Benchmark against the gas costs for comparable proof-verification
   precompiles on EVM networks where available (e.g. Ethereum's Groth16
   verifier, Arbitrum's curve pairing precompile).
3. Document end-to-end deposit latency (deposit submission to LEZ mint
   confirmation) and end-to-end redemption latency (burn to Ethereum release
   confirmation), each broken down by: chain finality wait, proof generation
   time, and on-chain verification time.
4. Document the RISC0 proof generation time for both inbound (Ethereum → LEZ)
   and outbound (LEZ → Ethereum) proofs, and characterize the compute resources
   (CPU, RAM, time) required to run a proof generator daemon.

#### Supportability

1. The Ethereum vault contract and the LEZ bridge program are deployed and
   tested on a public Ethereum testnet and LEZ devnet/testnet respectively.
2. End-to-end integration tests exercise the full deposit and redemption round
   trip against a LEZ sequencer (standalone mode) and an Ethereum test network
   or local fork, and are included in CI. CI must be green on the default
   branch.
3. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
4. A README documents end-to-end usage: contract and program addresses,
   deployment steps for both chains, and step-by-step instructions for
   depositing and redeeming via CLI and mini-app.
5. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for both deposit
   and redemption flows.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI and the proof generators (inbound and outbound), covering the
   core user and operator journeys respectively.
7. Provide Figma designs or equivalent for all mini-app GUI artefacts.
8. The Ethereum vault contract undergoes an independent third-party
   smart-contract security audit before mainnet deployment; the audit report
   (or a summary, if the full report is not publishable) must be linked from
   the README. This requirement exists because cross-chain bridges are the
   single most-attacked category of DeFi infrastructure (Chainalysis has
   tracked more than $2.8B stolen from bridges since 2022); it is not
   optional.
9. Provide a privacy and anonymisation properties document covering: what is
   visible on-chain for a deposit and a redemption on both Ethereum and LEZ;
   what remains private when minting to, or redeeming from, a LEZ private
   account; what the proof generators can observe about a depositor or
   redeemer from RPC queries to either chain; and what information is
   irretrievably obfuscated by the bridge's design.

#### + Bridge Security

1. RISC0 proof verification must be deterministic and verifiable by any
   participant independently. The LEZ bridge program and Ethereum vault contract
   must both reject proofs that fail verification, and this must be tested with
   invalid proofs (e.g. proofs with incorrect public inputs, proofs for incorrect
   chain state, proofs with tampered headers).
2. Proof generators must independently derive the deposit or burn tuple and chain
   state from RPC calls to the source chain; the daemon must not accept a proof
   supplied to it by a third party without independently regenerating and
   verifying it against the source chain.
3. Global and per-token deposit/redemption caps (Functionality #9) bound the
   maximum value at risk within any rolling window; proposals must document the
   recommended default caps and the reasoning behind them. These caps are
   independent of proof verification and operate as an additional administrative
   safety valve.
4. The freeze authority (Functionality #8) must be exercisable independently on
   the Ethereum vault and the LEZ bridge program, so that either half can be
   paused without requiring the other to be operational or reachable.
5. The bridge's user-facing documentation (mini-app, README, SDK docs) must
   state the trustless cryptographic verification model explicitly (see Design
   Rationale, "Trust model") and explain how RISC0 proofs ensure the integrity
   of cross-chain transfers without requiring trust in external signers.

### Soft Requirements

1. Batch proof verification: amortise verification cost across multiple deposits
   or redemptions in a single LEZ transaction, analogous to the multi-feed
   batching soft requirement in RFP-020.
2. Proof caching and relay: allow pre-generated and verified proofs to be cached
   and resubmitted by users or relayers, reducing the need for everyone to
   generate proofs independently (proof generators produce canonical proofs that
   can be reused).
3. A configurable per-token redemption delay (in addition to the finality wait)
   as an extra circuit-breaker window, allowing the freeze authority to react
   to an anomalous large redemption before funds leave the vault.
4. Support for wrapping ERC-20 tokens from additional EVM chains (e.g.
   Arbitrum, Base) behind the same LEZ bridge program, reusing the token
   registry and cap mechanism and generating separate RISC0 proofs for each
   source chain's consensus and state.
5. A path to improve proof generation over time: design the RISC0 proof
   generator as a pluggable component so that optimizations to the RISC0 circuit
   (e.g. via future RISC0 zkVM improvements, proof compression, or hardware
   acceleration) can be integrated without restructuring the vault or bridge
   program.

### Out of Scope

The following are explicitly excluded from this RFP:

- Wrapping non-fungible assets (ERC-721, ERC-1155).
- Bitcoin, Monero, and Zcash bridging: these have a dedicated trustless path
  via atomic swaps, delivered in [RFP-003](./RFP-003-atomic-swaps.md).
- Circuit optimization or custom RISC0 accelerators: proposals should leverage
  mature existing RISC0 implementations (e.g. Zisk) rather than implementing
  novel circuits or optimization techniques. Future improvements to RISC0
  circuit performance are a separate effort from this bridge implementation.
- Alternative proof systems (e.g. SNARKs, STARKs, other zkVMs): this RFP
  specifies RISC0 as the proof system. If future RISC0 versions or alternative
  systems become preferable, that is a candidate for a future update or new
  RFP.
- Price feeds for wrapped assets. Once a token is wrapped, pricing it (for
  lending collateral, DEX pairs, or the reflexive stablecoin) is the
  responsibility of the oracle stack
  ([RFP-019](./RFP-019-twap-oracle.md), [RFP-020](./RFP-020-redstone-oracle-adaptor.md)),
  not this bridge.

## ⚠ Platform Dependencies

### Hard dependencies

#### Admin authority (RFP-001)

The Functionality requirements specify that an admin authority registers and
deregisters supported tokens and configures deposit/redemption caps. These
admin-gated functions require the standardised admin authority library from
[RFP-001](./RFP-001-admin-authority-lib.md).

#### Freeze authority (RFP-002)

The Bridge Security requirements specify a freeze authority able to pause
minting and/or redemption, globally or per token, as a circuit breaker
independent of the deposit/redemption caps. This requires the standardised
freeze authority library from
[RFP-002](./RFP-002-freeze-authority-lib.md).

#### Token mint/burn authorities (LP-0013)

The LEZ bridge program mints wrapped tokens on verified deposits and burns them
on redemption. This requires the token mint/burn authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md).

#### RISC0 zkVM

The bridge verifies cryptographic proofs in-program (inbound on LEZ) and
natively on Ethereum (outbound). Proof generation and verification require
RISC0, a production-ready zkVM. Proposals must leverage mature RISC0
implementations (e.g. [Zisk](https://github.com/risc0/zisk)) rather than
building custom circuits.

### Soft dependencies

#### Event emission (LP-0012)

Redemption events, and ideally deposit-mint events, are emitted as structured
on-chain events so off-chain proof generators and dashboards can react
without polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
is **closed** (delivered).

## 👤 Recommended Team Profile

Team experienced with:

- Zero-knowledge proof systems and RISC0 zkVM (guest program development,
  proof generation and verification, public/private input handling)
- Solidity smart-contract development, including experience preparing a
  contract for third-party security audit, and EVM precompile integration
- Cryptographic primitives (Groth16 verification, Merkle proofs, consensus
  verification, state root inclusion proofs)
- LEZ program development and on-chain proof verification
- Smart-contract security auditing (proof validation, replay attacks,
  reorg handling, cap/rate-limit bypass)
- Cross-chain system design and integration testing (dual-chain atomic
  operations, finality assumptions, determinism and reproducibility)

## ⏱ Timeline Expectations

Estimated software delivery duration: **12–14 weeks**. This excludes the
third-party audit lead time required before mainnet deployment (Supportability
#8), which is typically procured and scheduled separately.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- [RFP-003 — Atomic Swaps](./RFP-003-atomic-swaps.md) (trustless path for BTC,
  XMR, ZEC; explicitly defers ETH to wrapping)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (primary consumer of wrapped collateral)
- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
  (consumer of wrapped-asset collateral)
- [RFP-019 — On-Chain TWAP Oracle](./RFP-019-twap-oracle.md) (assumes wrapped
  external assets as priced collateral)
- [RFP-020 — RedStone Off-Chain Oracle Adaptor for LEZ](./RFP-020-redstone-oracle-adaptor.md)
  (reference for in-program proof verification cost measurement)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
- [LP-0013: Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md)
- [RISC0 — Zero-Knowledge VM](https://github.com/risc0/risc0)
- [Zisk — RISC0 Proof Generation](https://github.com/risc0/zisk) (reference implementation for proof generation)
- [Chainalysis — Cross-Chain Bridge Hacks](https://www.chainalysis.com/blog/cross-chain-bridge-hacks-2022/)
  (bridge-hack loss data)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.