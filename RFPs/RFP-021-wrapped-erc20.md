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

Build a lock-and-mint bridge that lets ERC-20 tokens (and native ETH, wrapped as
WETH) held on Ethereum enter LEZ as canonical wrapped assets, and exit back to
Ethereum on redemption. An Ethereum-side vault contract escrows deposits; a
federation of attestors observes finalised deposits and threshold-signs an
attestation; a LEZ-side program verifies the threshold signature and mints the
corresponding wrapped token to the recipient's LEZ account. Redemption reverses
the flow: burning the wrapped token on LEZ triggers a federation-attested
release of the original ERC-20 from the vault. The verification core reuses the
in-program secp256k1 ECDSA + keccak256 path already built for
[RFP-020](./RFP-020-redstone-oracle-adaptor.md), so this RFP is primarily new
protocol wiring (vault contract, mint/burn program, federation daemon) rather
than new cryptographic groundwork. Teams should have experience with Solidity
smart-contract development, cross-chain relayer/federation architecture, and LEZ
program development.

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
possible instead, which is the standard architecture used to move value onto
every smart-contract chain that isn't Ethereum itself. It is simpler to build
than a trustless swap, but it is not trustless: it introduces a federation
trust assumption that this RFP must document honestly rather than obscure.

Stablecoins are the concrete prize. USDT and USDC together account for over 80%
of a stablecoin market that stood above $300B in mid-2026, and are the
settlement asset most DeFi money markets and DEXes actually run on. Without a
wrapped ERC-20 primitive, LEZ has no path to bring USDC, USDT, DAI, or WETH
liquidity onto the chain, which blocks the lending protocol and the reflexive
stablecoin from having any credible collateral base at launch.

Bridges are also the most attacked category of infrastructure in DeFi:
Chainalysis has tracked more than $2.8B stolen from cross-chain bridges since
2022, the highest-value class of exploit in the industry. This RFP's security
posture — a bounded, capped, admin-governed federation with a freeze-authority
circuit breaker, rather than an unbounded trust-me bridge — is designed
directly against that track record, and is explained in full in Design
Rationale and the Bridge Security requirements below.

## 🏗 Design Rationale

### Lock-and-mint via a federated attestor set

The bridge has two independently deployed halves that share one verification
primitive:

1. **Ethereum vault contract.** A Solidity contract that accepts deposits of any
   ERC-20 token in the supported-token registry (plus native ETH, which the
   vault wraps to WETH internally so ETH does not need a separate bridging
   path). Each deposit locks the token and emits a
   `Deposit(token, depositor, amount, lezRecipient, nonce)` event.
2. **Attestor federation.** An M-of-N set of attestors (default 3-of-5,
   configurable, registered and rotated by the admin authority per RFP-001)
   watches the Ethereum vault. Once a deposit reaches a configured finality
   depth, each attestor independently verifies the deposit and signs a
   secp256k1 ECDSA attestation over `(token, depositor, amount, lezRecipient,
   nonce)`.
3. **LEZ bridge program.** A public-mode LEZ program verifies the M-of-N
   threshold over the attestation — reusing the same in-program secp256k1 ECDSA
   + keccak256 verification path built for the RedStone adaptor
   ([RFP-020](./RFP-020-redstone-oracle-adaptor.md), Functionality #1–#2) — and
   mints the corresponding wrapped token to `lezRecipient`, crediting a private
   account directly where the recipient requests it (see "+ Privacy" below).
   The deposit `nonce` is recorded on first use and any repeat submission of the
   same nonce is rejected, so replaying an attestation cannot double-mint.

This is the same "signed attestation verified in-program" shape RFP-020 already
proved out for oracle data; the message being signed changes (a deposit tuple
instead of a price) but the verifier, the signer-set management, and the
M-of-N threshold logic are the same problem. Proposals are expected to share
code with, or directly depend on, the RFP-020 verification library rather than
reimplementing ECDSA recovery from scratch.

### Redemption: burn-and-release

Redemption is the mirror flow. A holder burns wrapped tokens on LEZ via the
bridge program, which validates the burn and emits a structured redemption
event (per [LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md))
carrying `(token, amount, ethereumRecipient, redemptionNonce)`. Once the burn
transaction reaches LEZ finality, the attestor federation observes the
redemption event and threshold-signs a release authorisation. The Ethereum
vault verifies the M-of-N signature — native `ecrecover` on the EVM, orders of
magnitude cheaper than the in-program RISC-V path, matching the asymmetry
RFP-020 already documented between LEZ-side and EVM-side ECDSA verification
cost — and releases the original ERC-20 to `ethereumRecipient`, tracking
`redemptionNonce` to reject replays on the Ethereum side.

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

### Trust model: federated custodial, not trustless

This bridge is **not** trustless. It composes with RFP-003's trustless BTC,
XMR, and ZEC swaps, not with the same security model: a user who bridges an
ERC-20 asset is trusting that at least `M` of the `N` attestors are honest and
that their keys are not simultaneously compromised. Proposals must state this
trust assumption plainly in user-facing documentation (mini-app, README,
SDK docs), rather than presenting the bridge as trust-minimised. Per-token and
global caps, the freeze-authority circuit breaker, and a documented attestor
rotation process (see Bridge Security) are the mitigations this RFP requires in
lieu of trustlessness; they bound the blast radius of a federation compromise,
they do not eliminate it. A fully trustless design — verifying Ethereum
consensus signatures or Merkle-Patricia state proofs inside the LEZ program —
is out of scope (see Out of Scope) and is a candidate for a future RFP once
in-circuit verification cost on RISC0 makes it practical, mirroring the
cost-conditional precompile path RFP-020 already laid out for its own
verification primitive.

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
2. Implement an attestor federation daemon (M-of-N, default 3-of-5) that
   observes finalised Ethereum deposits, verifies them independently, and
   signs a secp256k1 ECDSA attestation over `(token, depositor, amount,
   lezRecipient, nonce)`. Attestors must not sign before the configured
   Ethereum finality depth is reached.
3. Implement a public-mode LEZ bridge program that verifies the M-of-N
   attestation threshold (reusing the in-program secp256k1 ECDSA + keccak256
   verification path from [RFP-020](./RFP-020-redstone-oracle-adaptor.md)) and
   mints the corresponding wrapped token to `lezRecipient` — to a public or a
   private LEZ account, at the recipient's choice. Each deposit `nonce` may be
   consumed at most once; a repeat submission of an already-processed nonce
   must be rejected without minting.
4. Implement a burn instruction on the LEZ bridge program: a holder burns a
   specified amount of a wrapped token, and the program emits a structured
   redemption event (per LP-0012) carrying `(token, amount, ethereumRecipient,
   redemptionNonce)`.
5. Extend the attestor federation daemon to observe finalised LEZ redemption
   events and threshold-sign a release authorisation over `(token, amount,
   ethereumRecipient, redemptionNonce)`.
6. The Ethereum vault contract verifies the M-of-N release-authorisation
   signature and releases the original ERC-20 (or unwrapped ETH, for WETH
   redemptions) to `ethereumRecipient`. Each `redemptionNonce` may be consumed
   at most once.
7. An admin authority (per RFP-001, integrated via the SPEL framework where
   applicable to the LEZ side) can: register a new supported ERC-20 (Ethereum
   address, LEZ wrapped mint, decimals, per-token deposit/redemption caps),
   deregister a token, update the per-token caps, add or remove attestors, and
   change the M-of-N threshold. Registration and attestor-set changes must be
   mirrored consistently on both the Ethereum vault and the LEZ bridge program
   (document how the two sides stay in sync and what happens if they
   temporarily diverge).
8. A freeze authority (per RFP-002) can pause minting and/or redemption, either
   globally or for a single registered token, on both the Ethereum vault and
   the LEZ bridge program independently.
9. Global and per-token deposit and redemption caps (configurable by the admin
   authority) bound the maximum value that can be minted or released within a
   rolling window, as a rate-limiting circuit breaker independent of the
   freeze authority.

#### Usability

1. Provide an SDK that can be used to build Logos modules for: submitting
   deposits to the Ethereum vault, querying deposit/attestation/mint status,
   initiating a redemption (burn), querying redemption/release status, and
   reading the supported-token registry and current attestor set.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. The mini-app must
   cover: a deposit flow (connect an Ethereum wallet, approve and deposit,
   track attestation progress, confirm LEZ mint, choose public or private
   destination account), a redemption flow (burn wrapped tokens, track
   federation attestation, confirm Ethereum release), and a registry view
   (supported tokens, per-token caps and current utilisation, attestor set,
   current M-of-N threshold).
3. Provide a CLI that covers core functionality: deposit, query status, redeem,
   and query the token registry and attestor set. The CLI may have fewer
   features than the mini-app but must support all essential operations.
4. Provide the attestor federation daemon as a **Logos module accompanied by a
   Logos Core headless CLI/daemon**, so federation members can run it as a
   standalone long-running process. The daemon must support: configurable
   Ethereum and LEZ RPC endpoints, configurable finality depth for both
   chains, structured logging of observed deposits/redemptions and submitted
   attestations, signer-key management appropriate for production use (the
   daemon must not require the attestor's key to be stored in plaintext
   configuration), and a clean shutdown path. Document the attestor operator
   journey end-to-end: install, configure, run, monitor, rotate keys.
5. Provide an IDL for the LEZ bridge program using the
   [SPEL framework](https://github.com/logos-co/spel).
6. Return clear, actionable error messages for all failure modes: unsupported
   token, cap exceeded, attestation threshold not met, signer not in the
   authorised attestor set, insufficient finality, nonce already processed,
   and program or per-token frozen.

#### Reliability

1. Minting is atomic: a failed or rejected mint attempt does not consume the
   deposit nonce and leaves the deposit mintable on retry.
2. Burn-and-redeem is atomic: a failed redemption attempt does not burn tokens
   without a corresponding, retryable redemption record.
3. A deposit or redemption nonce can be consumed at most once; replaying an
   already-processed attestation or release authorisation is rejected
   deterministically and does not double-mint or double-release.
4. Federation liveness tolerates up to `N - M` attestors being offline or
   unresponsive without blocking deposits or redemptions.
5. A temporary RPC or connectivity failure on either chain leaves the attestor
   daemon in a recoverable state, able to resume observing and signing once
   connectivity is restored, without needing to replay already-attested items.

#### Performance

1. Verifying an M-of-N attestation (secp256k1 ECDSA + keccak256, in-program on
   LEZ) and minting must complete within a single LEZ public transaction at the
   per-transaction compute budget in force at delivery time. Document the
   compute-unit cost, extending the measurement methodology from
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md).
2. Document the Ethereum-side gas cost of verifying the M-of-N release
   authorisation and releasing funds from the vault, and compare it against
   the native-ECDSA cost range RFP-020 documents for EVM connectors (roughly
   50K–100K gas), since release-side verification runs as ordinary EVM
   `ecrecover`, not in a zkVM circuit.
3. Document end-to-end deposit latency (deposit submission to LEZ mint
   confirmation) and end-to-end redemption latency (burn to Ethereum release
   confirmation), each broken down by: chain finality wait, attestor
   observation and signing time, and on-chain verification time.

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
   for the CLI and the attestor daemon, covering the core user and operator
   journeys respectively.
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
   account; and what the attestor federation itself can observe about a
   depositor or redeemer regardless of the destination account type.

#### + Bridge Security

1. No single attestor can mint or release funds alone: the LEZ bridge program
   and the Ethereum vault contract must both enforce the configured M-of-N
   threshold, and this must be tested with fewer than `M` valid signatures
   present.
2. Attestor-set membership and the M-of-N threshold are updatable only by the
   admin authority (per RFP-001); the update path must keep the Ethereum vault
   and the LEZ bridge program's attestor sets consistent, and the update path
   itself must be tested, including a scenario where an attestor's key is
   suspected compromised and must be removed.
3. Global and per-token deposit/redemption caps (Functionality #9) bound the
   maximum value at risk from a compromised federation within any rolling
   window; proposals must document the recommended default caps and the
   reasoning behind them.
4. The freeze authority (Functionality #8) must be exercisable independently on
   the Ethereum vault and the LEZ bridge program, so that either half can be
   paused without requiring the other to be operational or reachable.
5. Attestors must independently re-derive the deposit or redemption tuple from
   on-chain data before signing; the daemon must not sign an attestation
   supplied to it by a third party without independently verifying it against
   the source chain.
6. The bridge's user-facing documentation (mini-app, README, SDK docs) must
   state the federated-custodial trust model explicitly (see Design
   Rationale, "Trust model") rather than describe the bridge as trustless.

### Soft Requirements

1. Batch attestation: amortise verification cost across multiple deposits or
   redemptions in a single LEZ transaction, analogous to the multi-feed
   batching soft requirement in RFP-020.
2. A configurable per-token redemption delay (in addition to the finality wait)
   as an extra circuit-breaker window, allowing the freeze authority to react
   to an anomalous large redemption before funds leave the vault.
3. Support for wrapping ERC-20 tokens from additional EVM chains (e.g.
   Arbitrum, Base) behind the same federation and program, reusing the token
   registry and cap mechanism.
4. A path to reduce the trust model over time: design the attestation
   verification as a swappable component (mirroring RFP-020's precompile
   swap-out requirement) so that a future light-client or ZK-proof-based
   Ethereum verification path can replace the federation without restructuring
   the vault or the LEZ bridge program.

### Out of Scope

The following are explicitly excluded from this RFP:

- A trustless bridge design that verifies Ethereum consensus signatures or
  Merkle-Patricia state proofs inside the LEZ program. This is a candidate for
  a future RFP once in-circuit verification cost on RISC0 makes it practical
  (see Design Rationale, "Trust model").
- Wrapping non-fungible assets (ERC-721, ERC-1155).
- Bitcoin, Monero, and Zcash bridging: these have a dedicated trustless path
  via atomic swaps, delivered in [RFP-003](./RFP-003-atomic-swaps.md).
- A governance- or DAO-based attestor selection process. The initial
  federation is admin-appointed per RFP-001; a permissionless or
  stake-weighted attestor set is a possible future evolution, not required
  here.
- Price feeds for wrapped assets. Once a token is wrapped, pricing it (for
  lending collateral, DEX pairs, or the reflexive stablecoin) is the
  responsibility of the oracle stack
  ([RFP-019](./RFP-019-twap-oracle.md), [RFP-020](./RFP-020-redstone-oracle-adaptor.md)),
  not this bridge.

## ⚠ Platform Dependencies

### Hard dependencies

#### Admin authority (RFP-001)

The Functionality requirements specify that an admin authority registers and
deregisters supported tokens, manages the attestor federation's membership and
threshold, and configures caps. These admin-gated functions require the
standardised admin authority library from
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

### Soft dependencies

#### Event emission (LP-0012)

Redemption events, and ideally deposit-mint events, are emitted as structured
on-chain events so the attestor daemon and off-chain dashboards can react
without polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
is **closed** (delivered).

#### Shared verification core (RFP-020)

The in-program secp256k1 ECDSA + keccak256 verification path built for the
RedStone oracle adaptor is directly reusable for verifying attestor signatures
here; the message format and signer-set model differ, but the recovery and
hashing primitives do not. This is a soft dependency: the bridge can ship with
its own verifier if RFP-020's library is not yet in a reusable state, but
proposals should default to sharing the audited implementation rather than
duplicating it.

## 👤 Recommended Team Profile

Team experienced with:

- Cross-chain bridge or relayer/federation architecture
- Solidity smart-contract development, including experience preparing a
  contract for third-party security audit
- Cryptographic verification (secp256k1 ECDSA recovery, keccak256 hashing,
  threshold/multi-signature schemes)
- LEZ / RISC0 program development
- Smart-contract security auditing (signer compromise, replay attacks,
  reorg handling, cap/rate-limit bypass)

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
  (source of the reusable secp256k1 ECDSA + keccak256 verification core)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
- [LP-0013: Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md)
- [Chainalysis — Cross-Chain Bridge Hacks](https://www.chainalysis.com/blog/cross-chain-bridge-hacks-2022/)
  (bridge-hack loss data)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.