---
id: RFP-021
title: Unlinkable Wrapped ERC-20 Bridge for LEZ
tier: L
status: open
category: Developer Tooling & Infrastructure
dependencies:
  - id: RFP-001
    reason: Admin authority governs the supported-token registry, the per-token denomination ladder, and the deposit/redemption caps, as specified in Functionality.
  - id: RFP-002
    reason: Freeze authority provides the circuit breaker to halt minting and/or redemption, globally or per token, if a circuit bug or vault vulnerability is suspected.
  - id: LP-0013
    reason: Mint/burn token authority primitives are required for the LEZ program to mint wrapped tokens into private state on verified deposit, and burn them on redemption.
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-021 — Unlinkable Wrapped ERC-20 Bridge for LEZ

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

Build a trustless, **unlinkable** lock-and-mint bridge that lets ERC-20 tokens
(and native ETH, wrapped as WETH) held on Ethereum enter LEZ as canonical
wrapped assets, and exit back to Ethereum on redemption — without any public
observer being able to connect a specific Ethereum lock to the LEZ mint it
funded, or a specific LEZ burn to the Ethereum unlock it triggered.

An Ethereum-side vault contract escrows deposits and publishes only a
*commitment* — never a LEZ recipient. A LEZ-side program trustlessly anchors the
Ethereum commitment-tree root by verifying a RISC0 proof of Ethereum consensus
and state in-program. A user then proves, entirely on their own device, that
they own an unspent leaf under that anchored root, and the program mints the
wrapped token into **private LEZ state**. Redemption mirrors this: a burn on LEZ
publishes only a withdrawal commitment — never an Ethereum recipient — and the
Ethereum vault, having anchored the LEZ burn-tree root via a RISC0 proof
verified through a native precompile, releases the original ERC-20 to whoever
proves ownership of an unspent leaf.

Nullifiers prevent double-spends on both legs without revealing which leaf was
consumed. Permissionless relayers submit the mint and unlock transactions so the
user never needs a funded account on either chain. The bridge core depends on
mature RISC0 implementations (e.g. Zisk) for consensus and state proof
generation, requiring teams experienced with zero-knowledge proof systems,
shielded-pool constructions, Solidity smart-contract development, LEZ program
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
cryptographic proofs natively.

### A transparent bridge would deanonymise the whole chain

Unlinkability is not a nice-to-have here; it is the difference between this
bridge strengthening LEZ's privacy guarantees and silently destroying them.

A conventional lock-and-mint bridge emits
`Deposit(token, depositor, amount, lezRecipient, nonce)` on Ethereum. That
single event publicly binds a KYC'd, fully-traced Ethereum address to a LEZ
account, permanently and for anyone to read. Because virtually all external
collateral enters LEZ through this one primitive, such a bridge becomes the
canonical deanonymisation oracle for the entire chain: an observer needs only to
scrape one Ethereum contract's logs to build an identity map covering most of
LEZ's collateral base. Every downstream privacy feature — private accounts,
the privacy-preserving DEX ([RFP-004](./RFP-004-privacy-preserving-dex.md)),
shielded lending positions — is undermined at the point of entry, no matter how
well those components protect data internally. Privacy that leaks at the on-ramp
is not privacy.

The redemption leg is symmetric and, if anything, worse: a burn event naming
`ethereumRecipient` in the clear publishes the exit address alongside the LEZ
account that funded it, closing the loop and linking a user's entire LEZ
activity to their Ethereum identity on both ends.

This RFP therefore treats unlinkability as a hard requirement of the same
standing as solvency. The construction is a commitment/nullifier shielded pool
on each leg, specified in full in Design Rationale below.

### Stablecoins are the concrete prize

USDT and USDC together account for over 80% of a stablecoin market that stood
above $300B in mid-2026, and are the settlement asset most DeFi money markets
and DEXes actually run on. Without a wrapped ERC-20 primitive, LEZ has no path
to bring USDC, USDT, DAI, or WETH liquidity onto the chain, which blocks the
lending protocol and the reflexive stablecoin from having any credible
collateral base at launch.

Bridges are also the most attacked category of infrastructure in DeFi:
Chainalysis has tracked more than $2.8B stolen from cross-chain bridges since
2022, the highest-value class of exploit in the industry. This RFP's security
posture — cryptographic verification via RISC0 zero-knowledge proofs,
eliminating the trust-in-signers vector entirely, combined with per-token and
global deposit/redemption caps and an admin-governed freeze-authority circuit
breaker — is designed directly against that track record. No signer compromise,
key theft, or participant collusion can lead to spurious minting or release.

## 🏗 Design Rationale

### The unlinkability requirement, stated precisely

Two properties must hold against an adversary who observes **all** public state
on both chains, indefinitely, and who may themselves deposit and redeem:

- **P1 (inbound).** Given an Ethereum lock event and the set of all LEZ mint
  transactions, the adversary cannot determine which mint consumed that lock
  with probability better than uniform over the inbound anonymity set.
- **P2 (outbound).** Given a LEZ burn event and the set of all Ethereum unlock
  transactions, the adversary cannot determine which unlock was funded by that
  burn with probability better than uniform over the outbound anonymity set.

Both properties are stated relative to an *anonymity set*, and both degrade to
nothing when that set is small. Sizing, measuring and surfacing the anonymity
set is therefore a first-class requirement, not an implementation detail — see
Usability #7 and the Privacy & Unlinkability requirements.

Two facts are fixed by the environment and cannot be designed away. Proposals
must not claim otherwise:

1. **The Ethereum lock amount is public.** It is an ERC-20 `Transfer` into the
   vault. So is the depositor's address, since they send the transaction.
2. **The Ethereum unlock amount and recipient are public.** The vault must move
   real tokens to a real address.

Unlinkability is achieved by making these public facts *uninformative about
which counterparty they pair with*, not by hiding them.

### Fixed denominations

Amounts are the strongest linkage vector: a lock of 1,337.42 USDC followed by a
mint of 1,337.42 wUSDC is linked by elimination regardless of what cryptography
sits between them. Since the Ethereum-side amount cannot be hidden, its
*fingerprint* must be destroyed instead.

Every supported token therefore carries an admin-registered **denomination
ladder** (e.g. 1 / 10 / 100 / 1,000 / 10,000 USDC). Locks and unlocks are
permitted only at exactly these values; a user bridging 4,300 USDC does so as
4×1,000 + 3×100 notes. Thousands of users share each rung, so the public amount
identifies only the rung, not the user. Supply accounting remains scalar, which
keeps the LEZ mint authority within what [LP-0013](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md)
provides, and keeps the caps in Functionality #11 straightforwardly enforceable.

The cost is real and must be handled in the UX: users split large transfers
across several notes, and remainders below the smallest rung cannot be bridged.
Proposals must specify the default ladder per token and justify the rung
spacing against expected transfer sizes.

Shielded value commitments — hiding amounts entirely via homomorphic supply
accounting and range proofs — are the stronger construction and are specified as
Soft Requirement #1. The note format, circuit interfaces, and tree layout
delivered under this RFP must be chosen so that migration to shielded values
does not require redeploying the vault or re-anchoring the pool.

### Inbound: lock → anchor → prove → mint into private state

1. **Ethereum vault contract.** A Solidity contract holding an incremental
   Merkle tree of deposit commitments. To deposit, a user picks a denomination
   `d` for a registered token, samples secrets locally, and computes
   `C = H(nullifierSecret, spendSecret, token, d)`. The vault pulls `d` units of
   the token and inserts `C` as a leaf. The emitted event is
   `Deposit(token, d, commitment, leafIndex, noteCiphertext)`.

   **There is no `lezRecipient` field and no `depositor` field.** This is the
   central change from a conventional bridge. `noteCiphertext` is the note
   encrypted to the recipient's LEZ viewing key, allowing a third-party
   depositor to fund someone else's LEZ account without an out-of-band secret
   handoff; the recipient discovers it by trial decryption.

2. **Trustless root anchoring on LEZ.** A permissionless daemon submits a RISC0
   proof of (a) Ethereum consensus at a finalised block height and (b) the value
   of the vault's commitment-tree root in that state. The LEZ bridge program
   verifies this proof **in-program** and records the root. The daemon handles
   only public chain data, holds no user secrets, and cannot insert a false root
   — the program rejects any root not backed by a valid consensus proof. This is
   what makes client-side proving feasible without introducing a trusted party.

3. **Client-side ownership proof.** The user's own device generates a proof that
   it knows the secrets behind *some* leaf under an anchored root, and derives
   `nullifier = H(nullifierSecret)`. Public inputs are
   `(anchoredRoot, token, d, nullifier, relayer, relayerFee)`. The Merkle path,
   the leaf index, and both secrets are private inputs and never leave the
   device. Because the proof is only a membership-and-hash argument against an
   already-anchored root — not a consensus verification — it is small enough to
   generate on the desktop hardware Basecamp already targets.

4. **Mint.** The LEZ bridge program verifies the proof, rejects the transaction
   if `nullifier` is already in the spent set, records it, and mints `d` units
   of the wrapped token — `relayerFee` to the relayer's public account and the
   remainder **into private LEZ state**, where neither the balance nor the owner
   is publicly visible.

The mint transaction publishes a nullifier and a root. It reveals nothing about
which of the N unconsumed same-denomination deposits it consumed, satisfying P1
with an anonymity set of N.

### Outbound: burn → commit → anchor → prove → unlock

The redemption leg is the inbound leg run backwards, and its critical property
is that **the burn does not name the Ethereum recipient**. Naming it there is
precisely the leak that P2 forbids, so the flow is split into two stages
separated by a user-chosen delay.

1. **Burn on LEZ.** A holder spends a private wrapped-token note of value `d`,
   samples a fresh `ethSecret` locally, and the bridge program inserts a
   withdrawal commitment `W = H(ethSecret, token, d)` into an on-chain
   burn-commitment tree, emitting a structured event (per
   [LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md))
   carrying `(token, d, W, leafIndex)` — and nothing else. The spent LEZ note's
   own nullifier is recorded to prevent double-burn.

2. **Trustless root anchoring on Ethereum.** A permissionless daemon submits a
   RISC0 proof of LEZ consensus at a finalised epoch and the burn-tree root in
   that state. The vault verifies it via a **native precompile** (Groth16
   verifier or equivalent) — a single cheap verification step, matching the
   LEZ-side/EVM-side cost asymmetry documented in RFP-020 — and records the
   root.

3. **Client-side unlock proof.** At a time of the user's choosing, their device
   proves knowledge of `ethSecret` for some leaf under an anchored burn root,
   publishing `nullifier' = H(ethSecret)` and public inputs
   `(anchoredBurnRoot, token, d, ethereumRecipient, relayer, relayerFee)`.

4. **Release.** The vault verifies the proof, rejects a replayed `nullifier'`,
   and releases `d - relayerFee` of the original ERC-20 (or unwrapped ETH, for
   WETH redemptions) to `ethereumRecipient`, and `relayerFee` to the relayer.

`ethereumRecipient` is bound inside the proof, so a relayer can neither redirect
the funds nor inflate their fee. The unlock reveals a recipient and an amount,
but not which burn funded it, satisfying P2.

### Client-side proving, and why no daemon holds a witness

All proving that touches user-linked data happens on the user's device. The
daemons exist only to anchor roots, and a root is public chain state — a daemon
that observed every deposit and every anchor operation still cannot say which
mint consumed which lock, because it never sees a witness.

Critically, the daemons are also **trustless, not merely honest-but-curious**:
each anchoring submission carries a consensus proof that the receiving chain
verifies before accepting the root. A malicious or compromised daemon can
withhold service, but cannot forge a root, cannot mint, and cannot deanonymise.
Anchoring is permissionless precisely so that withholding is not a viable
attack — any party can run a daemon, and proposals must document how a user can
run their own or trigger an anchor themselves if none is available.

Proposals must state explicitly, in the privacy document required by
Supportability #9, that no component other than the user's own client ever
receives a Merkle path, a leaf index, or a note secret.

### Timing decorrelation

A perfect commitment scheme is defeated by a user who locks on Ethereum and
mints on LEZ ninety seconds later when no one else is bridging. Timing is a
linkage vector in its own right and the protocol must not force users into
correlated behaviour.

Accordingly: a proof may be submitted at **any** time after the relevant root is
anchored, with no expiry; the client must default to a randomised delay rather
than immediate submission; and the client must show the user the current
anonymity set for their `(token, denomination)` pair and warn before submitting
into a set below a configurable threshold. Users must be able to override, but
must be told what they are giving up.

### Relayers and fee payment

If a user pays their own LEZ gas to mint, the funding source of that account
re-links them to the deposit, and the whole construction collapses. Mint and
unlock transactions are therefore submitted by **permissionless relayers**, paid
out of the bridged amount inside the circuit.

Because `relayer` and `relayerFee` are public inputs bound into the proof, a
relayer cannot alter the recipient, redirect funds, or claim more than the
agreed fee; the worst they can do is decline to relay. Fee amounts must be drawn
from a small fixed schedule (or a fixed fraction of the denomination) rather
than being freely user-chosen — an arbitrary fee value is itself a fingerprint
that re-identifies the transaction.

### Note custody and recovery

In a commitment scheme, losing a note secret means the locked ERC-20 is
permanently unrecoverable — there is no administrative recovery path, by
construction. This is the single largest UX risk in the design and must be
addressed directly, not disclaimed.

Note secrets must therefore be derived deterministically from a single
recoverable seed (e.g. a signature over a fixed domain-separated message from
the user's existing wallet), so that a user restoring that wallet can rediscover
all their notes by scanning the commitment trees and trial-decrypting note
ciphertexts. Ad-hoc random secrets requiring manual backup are not acceptable.

### Token registry and decimal normalisation

Each supported ERC-20 is registered individually by the admin authority: its
Ethereum contract address, its LEZ wrapped-token mint, its decimals, its
denomination ladder, and its per-token deposit/redemption caps. ERC-20 tokens
do not share a common decimals convention (6 for USDC, 8 for WBTC, 18 for WETH
and DAI); the wrapped LEZ mint for each token must document its own decimals and
the exact conversion applied on mint and burn, and the registration instruction
must reject a token whose decimals or denomination ladder cannot be represented
exactly in the chosen LEZ mint precision.

Fee-on-transfer and rebasing ERC-20s break the invariant that a locked
denomination equals a mintable denomination. The registry must reject them, and
the vault must verify the actual balance delta on deposit rather than trusting
the requested amount.

### Finality windows and reorg protection

A root must not be anchored until the source-chain state it commits to has
reached a configured finality depth (a block-count or, once generally available,
a beacon-chain finalised-checkpoint condition). Anchoring a pre-finality root
risks admitting deposits that a reorg later removes, which would mint wrapped
tokens against locks that no longer exist. The chosen depth is a direct
trade-off between user-facing latency and reorg risk, and proposals must
document the depth chosen and the residual risk it leaves. The reverse direction
is symmetric: the vault anchors a LEZ burn root only after LEZ finality.

Because anchoring is decoupled from user proofs, a reorg affects only the
anchoring step. Anchored roots are append-only and never rolled back, so no user
proof is ever invalidated by later chain activity.

### Trust model: cryptographic, trustless

This bridge is **trustless**. It requires no trust in external signers,
validators, attestor federations, or relayers. A user who bridges an ERC-20
asset relies only on the correctness of the RISC0 circuits, the security of the
Ethereum network, and the security of the LEZ network. Both legs are verified by
cryptographic proofs, not by signatures from a bounded set of participants.

Relayers are trusted for **liveness only** — they can decline to submit, but
cannot steal, redirect, censor selectively (they cannot tell users apart), or
deanonymise. Anchoring daemons are likewise liveness-only.

Proposals must state this trustlessness explicitly in user-facing documentation
(mini-app, README, SDK docs), alongside an honest statement of the
unlinkability guarantee and its dependence on anonymity-set size. Per-token and
global caps and the freeze-authority circuit breaker remain in place as
operational safety mechanisms independent of the cryptographic trust model.

Note that the freeze authority operates at token and protocol granularity only.
It **cannot** freeze an individual user's funds, because the protocol does not
know which funds belong to whom. This is a designed consequence of
unlinkability, not a gap, and must be documented as such.

### Fee structure

This RFP does not mandate a specific protocol fee rate (relayer fees are
separate and market-set within the fixed schedule above). Proposals must specify
who pays, when fees are collected, the exact rate, and where fees are routed.
A governance-activatable fee switch with an initial zero rate, gated by the
admin authority per RFP-001, is the recommended baseline, consistent with the
pattern used elsewhere in the Logos RFP set (see
[RFP-017](./RFP-017-token-vesting.md), "Fee structure"). Any protocol fee must
be a fixed function of the denomination, for the fingerprinting reason above.

## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

1. Implement an Ethereum vault contract (Solidity) maintaining an incremental
   Merkle tree of deposit commitments. It accepts deposits of any ERC-20 in the
   supported-token registry, plus native ETH (auto-wrapped to WETH), **only at
   values on that token's registered denomination ladder**. It emits
   `Deposit(token, denomination, commitment, leafIndex, noteCiphertext)` and
   **must not** emit, store, or otherwise publish a LEZ recipient. The vault
   verifies the actual balance delta on deposit and rejects any token whose
   transfer does not deliver the exact denomination.
2. Implement a permissionless inbound anchoring path: a RISC0 proof of Ethereum
   consensus at a configured finality depth plus the vault's commitment-tree
   root in that state, verified **in-program** by the LEZ bridge program before
   the root is recorded. Anchored roots are append-only. Submitting an
   unbacked, malformed, or pre-finality root must be rejected without recording.
3. Implement a public-mode LEZ bridge program mint instruction accepting a
   client-generated ownership proof with public inputs
   `(anchoredRoot, token, denomination, nullifier, relayer, relayerFee)`. On
   successful verification it mints `relayerFee` to the relayer's public account
   and the remainder **into private LEZ state**. The Merkle path, leaf index,
   and note secrets must be private inputs to the circuit.
4. Each inbound `nullifier` may be consumed at most once; resubmission of a
   spent nullifier must be rejected deterministically without minting, and must
   not reveal which leaf the nullifier corresponds to.
5. Implement a burn instruction on the LEZ bridge program: a holder spends a
   private wrapped-token note of a registered denomination, and the program
   inserts a withdrawal commitment into an on-chain burn-commitment tree and
   emits a structured event (per LP-0012) carrying
   `(token, denomination, withdrawalCommitment, leafIndex)`. The instruction and
   its event **must not** accept, store, or publish an Ethereum recipient
   address.
6. Implement a permissionless outbound anchoring path: a RISC0 proof of LEZ
   consensus at a configured finality depth plus the burn-tree root in that
   state, verified by the Ethereum vault via a native precompile (Groth16
   verifier or equivalent) before the root is recorded. Anchored roots are
   append-only.
7. The Ethereum vault accepts a client-generated unlock proof with public inputs
   `(anchoredBurnRoot, token, denomination, ethereumRecipient, relayer, relayerFee)`,
   verifies it via the precompile, and releases `denomination - relayerFee` of
   the original ERC-20 (or unwrapped ETH, for WETH redemptions) to
   `ethereumRecipient` and `relayerFee` to `relayer`. Each outbound nullifier
   may be consumed at most once.
8. Note secrets are derived deterministically from a single recoverable seed.
   Implement note discovery: given the seed, a client can reconstruct its full
   note set by scanning both commitment trees and trial-decrypting note
   ciphertexts, recovering all unspent notes without any server-side index.
9. `relayerFee` must be constrained on-chain to a value on a registered fee
   schedule (or a fixed fraction of the denomination); a proof carrying an
   arbitrary fee value must be rejected.
10. An admin authority (per RFP-001, integrated via the SPEL framework where
    applicable to the LEZ side) can register a new supported ERC-20 (Ethereum
    address, LEZ wrapped mint, decimals, denomination ladder, fee schedule,
    per-token caps) and deregister a token. Registration changes must be
    mirrored consistently on both sides (document how the two stay in sync and
    what happens if they temporarily diverge). The registry must reject
    fee-on-transfer and rebasing tokens.
11. Global and per-token deposit and redemption caps (configurable by the admin
    authority) bound the maximum value that can be minted or released within a
    rolling window, as a rate-limiting circuit breaker independent of the freeze
    authority. Cap enforcement must not require identifying individual users.
12. A freeze authority (per RFP-002) can pause minting and/or redemption, either
    globally or for a single registered token, on both the Ethereum vault and
    the LEZ bridge program independently.

#### Usability

1. Provide an SDK to build Logos modules for: constructing and submitting
   deposits, deriving and managing notes, discovering owned notes by scanning,
   generating inbound and outbound proofs locally, submitting via a relayer,
   querying anchoring progress and anonymity-set size, initiating a redemption
   burn, and reading the supported-token registry and denomination ladders.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. It must cover: a
   deposit flow (connect Ethereum wallet, select denomination, approve and
   deposit, note derivation and backup confirmation), a mint flow (local proof
   generation with progress, delay selection, relayer submission), a redemption
   flow (burn, then a separately-timed unlock with recipient entry), a note
   inventory and recovery flow, and a registry view (supported tokens, ladders,
   caps and current utilisation).
3. Provide a CLI covering core functionality: deposit, discover notes, generate
   proof, mint, burn, unlock, and query the registry. The CLI may have fewer
   features than the mini-app but must support all essential operations,
   including proof generation and note recovery entirely offline.
4. Provide the inbound and outbound anchoring daemons as **Logos modules
   accompanied by Logos Core headless CLI/daemons**, runnable as standalone
   long-running processes. Each must support: configurable Ethereum and/or LEZ
   RPC endpoints, configurable finality depth, configurable anchoring cadence,
   structured logging, and a clean shutdown path. They must integrate with or
   wrap mature RISC0 implementations (e.g. Zisk) without reimplementing
   zero-knowledge primitives. Document the operator journey end-to-end: install,
   configure, run, monitor.
5. Provide a reference relayer implementation, runnable permissionlessly, with
   documented fee policy and submission logic. Document how a user submits
   without a relayer if they accept the privacy cost of paying their own gas.
6. Provide an IDL for the LEZ bridge program using the
   [SPEL framework](https://github.com/logos-co/spel).
7. Both the mini-app and CLI must display, before any mint or unlock
   submission, the current anonymity set size for the relevant
   `(token, denomination)` pair, and must warn when it falls below a
   configurable threshold. The default submission delay must be randomised
   rather than immediate, with the user able to inspect and override it.
8. Return clear, actionable error messages for all failure modes: unsupported
   token, invalid denomination, cap exceeded, invalid proof, proof verification
   failure, root not yet anchored, insufficient finality, nullifier already
   spent, invalid relayer fee, and program or per-token frozen. Error messages
   must not leak which leaf or note a failed proof referred to.

#### Reliability

1. Minting is atomic: a failed or rejected mint attempt does not consume the
   nullifier and leaves the note spendable on retry.
2. Burn-and-unlock is atomic at each stage: a failed burn does not destroy a
   note without inserting a corresponding withdrawal commitment, and a failed
   unlock does not consume the outbound nullifier.
3. A nullifier can be consumed at most once on each leg; replaying an
   already-verified proof is rejected deterministically and does not double-mint
   or double-release.
4. Anchored roots are append-only and never invalidated. A proof valid against a
   historical anchored root must remain valid indefinitely, so a user who
   delays submission for privacy reasons is never forced to regenerate.
5. Note recovery is complete: a client restored from seed alone must rediscover
   every unspent note it owns on both legs, verified by a test that wipes all
   local state and recovers from seed.
6. A temporary RPC or connectivity failure leaves an anchoring daemon in a
   recoverable state, able to resume once connectivity is restored without
   re-anchoring already-anchored roots.
7. Client-side proof generation is resumable or cheaply restartable: an
   interrupted proof does not consume, corrupt, or expose the note.

#### Performance

1. Verifying an inbound ownership proof and minting must complete within a
   single LEZ public transaction at the per-transaction compute budget in force
   at delivery time. Document the compute-unit cost, broken down between proof
   verification, nullifier-set insertion, and the mint instruction, extending
   the measurement methodology from
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md).
2. Document the compute-unit cost of in-program verification of the inbound
   **anchoring** proof (Ethereum consensus + state), separately from #1, and the
   amortised per-deposit cost at the recommended anchoring cadence.
3. Document the Ethereum-side gas cost of (a) a deposit, (b) verifying an
   outbound anchoring proof via precompile, and (c) verifying an unlock proof
   and releasing funds. Benchmark against comparable proof-verification
   precompiles on EVM networks where available.
4. **Client-side proof generation must be practical on the desktop hardware
   Basecamp runs on.** Measure and document wall-clock generation time and peak
   memory for both the inbound ownership proof and the outbound unlock proof, on
   at least: a mid-range laptop, and the lowest-specification machine the team
   declares as supported. State that minimum specification explicitly, and
   report results for the mini-app and the CLI separately if their proving paths
   differ. Mobile and browser proving targets are out of scope (see Out of
   Scope).
5. Document end-to-end deposit latency (lock to LEZ mint) and redemption latency
   (burn to Ethereum release), each broken down by: source-chain finality wait,
   anchoring wait, client proof generation, privacy delay, and on-chain
   verification.
6. Document anchoring proof generation time for both directions and the compute
   resources (CPU, RAM, time) required to run an anchoring daemon.
7. Document the growth rate and on-chain storage cost of both commitment trees
   and both nullifier sets, and the projected cost at 1M and 10M notes.

#### Supportability

1. The Ethereum vault contract and the LEZ bridge program are deployed and
   tested on a public Ethereum testnet and LEZ devnet/testnet respectively.
2. End-to-end integration tests exercise the full deposit and redemption round
   trip against a LEZ sequencer (standalone mode) and an Ethereum test network
   or local fork, and are included in CI. CI must be green on the default
   branch.
3. Every hard requirement in Functionality, Usability, Reliability, Performance,
   and Privacy & Unlinkability has at least one corresponding test.
4. A README documents end-to-end usage: contract and program addresses,
   deployment steps for both chains, and step-by-step instructions for
   depositing, minting, burning and unlocking via CLI and mini-app.
5. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for both legs
   including note management and local proof generation.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, the anchoring daemons, and the reference relayer, covering the
   core user and operator journeys respectively.
7. Provide Figma designs or equivalent for all mini-app GUI artefacts, including
   the anonymity-set disclosure and the note-backup confirmation flows.
8. The Ethereum vault contract undergoes an independent third-party
   smart-contract security audit before mainnet deployment; the audit report (or
   a summary, if the full report is not publishable) must be linked from the
   README. This requirement exists because cross-chain bridges are the single
   most-attacked category of DeFi infrastructure (Chainalysis has tracked more
   than $2.8B stolen from bridges since 2022); it is not optional.
9. Provide a **privacy and unlinkability properties document** covering: a
   formal statement of P1 and P2 and the anonymity set each is measured against;
   exactly what is visible on-chain for every operation on both chains; what an
   adversary observing all public state can and cannot infer; confirmation that
   no component other than the user's client ever receives a Merkle path, leaf
   index, or note secret; what a relayer, an anchoring daemon operator, and an
   RPC provider can each observe; residual leakage from timing, denomination
   choice, gas payment, IP-level metadata, and note-splitting patterns; and the
   conditions under which unlinkability degrades or fails.
10. Document the anonymity-set growth model: expected set size over time at
    projected volumes, the minimum set size below which the guarantee is
    considered not to hold, and guidance for users bridging before the pool has
    matured.

#### + Bridge Security

1. RISC0 proof verification must be deterministic and independently verifiable.
   The LEZ program and Ethereum vault must both reject proofs that fail
   verification, tested with invalid proofs (incorrect public inputs, proofs for
   incorrect chain state, tampered headers, proofs against unanchored roots,
   proofs reusing a spent nullifier).
2. Anchoring daemons must independently derive chain state from RPC calls to the
   source chain; a daemon must not accept and forward a root or proof supplied
   by a third party without independently regenerating and verifying it.
3. A malicious relayer must not be able to alter `ethereumRecipient`, inflate
   `relayerFee`, or replay a user's proof to a different destination. Test each
   case explicitly.
4. Global and per-token caps (Functionality #11) bound the maximum value at risk
   in any rolling window; proposals must document recommended default caps and
   the reasoning behind them.
5. The freeze authority (Functionality #12) must be exercisable independently on
   each half, so either can be paused without the other being operational or
   reachable.
6. Soundness of supply: the total wrapped supply on LEZ must never exceed the
   vault's holdings. Provide a test that attempts to mint without a valid
   deposit, mint twice from one deposit, and unlock without a valid burn.
7. User-facing documentation must state the trustless verification model and the
   liveness-only role of relayers and anchoring daemons (see Design Rationale,
   "Trust model").

#### + Privacy & Unlinkability

1. **P1 must hold under test.** Provide an automated test that constructs a
   population of deposits and mints and asserts that no public-state-derived
   correlation identifies the true lock↔mint pairing better than chance across
   the anonymity set.
2. **P2 must hold under test.** The equivalent test for burn↔unlock pairings.
3. No deposit event, burn event, transaction argument, log, or account-state
   change on either chain may contain a LEZ recipient at deposit time or an
   Ethereum recipient at burn time. Provide a test that asserts this over full
   event and state diffs for a complete round trip.
4. Merkle paths, leaf indices, and note secrets must be private circuit inputs.
   Provide a test asserting they are absent from all public inputs and all
   submitted transaction data.
5. Failure and error paths must not leak note identity: a rejected proof, a
   spent nullifier, and a cap rejection must be indistinguishable with respect
   to which leaf was involved.
6. The client must not contact any network endpoint that reveals which note it
   is proving over. Document all network calls made during proof generation and
   justify each; a note-specific query to a third-party RPC is a defect.
7. Relayer selection and submission must not create a linkage channel: document
   what a relayer learns, and ensure a user can switch relayers per operation.
8. The default configuration must be the private one. No user action should be
   required to obtain unlinkability, and any override that weakens it must
   require explicit confirmation.

### Soft Requirements

1. **Shielded value commitments.** Replace fixed denominations with hidden
   amounts via Pedersen or equivalent value commitments, range proofs, and
   join-split notes, with homomorphic supply accounting on the LEZ mint
   authority. This removes the denomination ladder entirely and merges all
   per-rung anonymity sets into one. The note format, circuit interfaces, and
   tree layout delivered under the hard requirements must be designed so this
   migration does not require redeploying the vault or re-anchoring the pool;
   document the intended migration path even if it is not implemented.
2. Batch verification: amortise proof verification across multiple mints or
   unlocks in a single transaction, analogous to the multi-feed batching soft
   requirement in RFP-020. Batching also improves unlinkability by making
   individual operations harder to isolate.
3. Optional viewing keys allowing a user to *voluntarily* disclose their own
   bridge activity to a chosen third party, without weakening unlinkability for
   anyone else and without any protocol-level disclosure capability.
4. A configurable per-token unlock delay (in addition to finality and the user's
   own privacy delay) as an extra circuit-breaker window, allowing the freeze
   authority to react to anomalous redemption volume before funds leave the
   vault.
5. Support for wrapping ERC-20 tokens from additional EVM chains (e.g. Arbitrum,
   Base) behind the same LEZ bridge program, reusing the registry, ladders, and
   caps, with separate anchoring paths per source chain. Consider whether pools
   should be shared across source chains to enlarge anonymity sets.
6. Hardware-accelerated or GPU-assisted client proving as an optional path for
   users with capable hardware, without making it a requirement.
7. Design the RISC0 anchoring components as pluggable so that future zkVM
   improvements, proof compression, or hardware acceleration can be integrated
   without restructuring the vault, the bridge program, or the note format.

### Out of Scope

The following are explicitly excluded from this RFP:

- Wrapping non-fungible assets (ERC-721, ERC-1155).
- Bitcoin, Monero, and Zcash bridging: these have a dedicated trustless path via
  atomic swaps, delivered in [RFP-003](./RFP-003-atomic-swaps.md).
- Mobile and in-browser proving. Basecamp is a desktop application and is the
  delivery surface for the mini-app, so the prover targets desktop only.
  Proposals must not constrain circuit design to fit mobile-class resource
  budgets.
- Network-level anonymity. Unlinkability here is a property of on-chain state.
  IP-level correlation between a user's Ethereum deposit and their relayer
  submission is out of scope as an implementation concern, but must be disclosed
  as residual leakage under Supportability #9.
- Protocol-level compliance, disclosure, or selective-deanonymisation
  mechanisms. Voluntary user-held viewing keys are Soft Requirement #3; any
  capability allowing a third party to deanonymise a user without their consent
  is contrary to the design and out of scope.
- Circuit optimization or custom RISC0 accelerators: proposals should leverage
  mature existing RISC0 implementations (e.g. Zisk) rather than implementing
  novel circuits or optimization techniques.
- Alternative proof systems (e.g. other zkVMs): this RFP specifies RISC0. If
  future RISC0 versions or alternative systems become preferable, that is a
  candidate for a future update or new RFP.
- Price feeds for wrapped assets. Once a token is wrapped, pricing it is the
  responsibility of the oracle stack
  ([RFP-019](./RFP-019-twap-oracle.md), [RFP-020](./RFP-020-redstone-oracle-adaptor.md)),
  not this bridge.

## ⚠ Platform Dependencies

### Hard dependencies

#### Admin authority (RFP-001)

The Functionality requirements specify that an admin authority registers and
deregisters supported tokens and configures denomination ladders, fee schedules
and caps. These admin-gated functions require the standardised admin authority
library from [RFP-001](./RFP-001-admin-authority-lib.md).

#### Freeze authority (RFP-002)

The Bridge Security requirements specify a freeze authority able to pause
minting and/or redemption, globally or per token, as a circuit breaker
independent of the caps. This requires the standardised freeze authority library
from [RFP-002](./RFP-002-freeze-authority-lib.md).

#### Token mint/burn authorities (LP-0013)

The LEZ bridge program mints wrapped tokens into private state on verified
deposits and burns them on redemption. This requires the token mint/burn
authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md).
The fixed-denomination design keeps supply accounting scalar and therefore
within what LP-0013 provides; Soft Requirement #1 (shielded values) would
require homomorphic supply accounting beyond it, and proposals pursuing that
path must state what extension is needed.

#### Private LEZ account state

Inbound mints deliver into private LEZ state, and outbound burns spend from it.
Unlinkability depends on this: a mint into a public account re-links the
recipient immediately. Proposals must state which LEZ private-state primitives
they rely on and their maturity.

#### RISC0 zkVM

The bridge verifies consensus proofs in-program (inbound on LEZ) and natively on
Ethereum (outbound), and generates ownership proofs client-side. This requires
RISC0, a production-ready zkVM. Proposals must leverage mature RISC0
implementations (e.g. [Zisk](https://github.com/risc0/zisk)) rather than
building custom circuits.

### Soft dependencies

#### Event emission (LP-0012)

Burn and anchoring events are emitted as structured on-chain events so clients
and anchoring daemons can react without polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
is **closed** (delivered).

## 👤 Recommended Team Profile

Team experienced with:

- Zero-knowledge proof systems and RISC0 zkVM (guest program development, proof
  generation and verification, public/private input handling)
- **Shielded-pool constructions**: commitment/nullifier schemes, incremental
  Merkle trees, note encryption and discovery, deterministic key derivation, and
  the anonymity-set analysis that goes with them
- Solidity smart-contract development, including experience preparing a contract
  for third-party security audit, and EVM precompile integration
- Cryptographic primitives (Groth16 verification, Merkle proofs, consensus
  verification, state root inclusion proofs)
- LEZ program development, private-state programs, and on-chain proof
  verification
- Client-side proving on desktop targets, including packaging a prover inside a
  Basecamp mini-app and a headless CLI
- Smart-contract security auditing (proof validation, replay attacks, reorg
  handling, cap/rate-limit bypass, privacy-leak analysis)
- Cross-chain system design and integration testing (dual-chain atomic
  operations, finality assumptions, determinism and reproducibility)

## ⏱ Timeline Expectations

Estimated software delivery duration: **16–20 weeks**. This is longer than a
transparent lock-and-mint bridge would require; the shielded-pool construction,
client-side proving targets, note discovery and recovery, the relayer layer, and
the unlinkability test suite are the additional scope. This excludes the
third-party audit lead time required before mainnet deployment (Supportability
#8), which is typically procured and scheduled separately.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- [RFP-003 — Atomic Swaps](./RFP-003-atomic-swaps.md) (trustless path for BTC,
  XMR, ZEC; explicitly defers ETH to wrapping)
- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (downstream consumer whose privacy guarantees depend on a private on-ramp)
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
