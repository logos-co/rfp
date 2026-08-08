---
id: RFP-021
title: Privacy-Preserving Wrapped ERC-20 Bridge for LEZ
tier: L
status: open
category: Developer Tooling & Infrastructure
dependencies:
  - id: RFP-001
    reason: Admin authority governs the supported-token registry and the deposit/redemption caps, as specified in Functionality.
  - id: RFP-002
    reason: Freeze authority provides the circuit breaker to halt minting and/or redemption, globally or per token, if a proof-system or vault vulnerability is suspected.
  - id: LP-0013
    reason: Mint/burn token authority primitives are required for the LEZ program to mint wrapped tokens on verified deposit, and burn them on redemption.
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-021 — Privacy-Preserving Wrapped ERC-20 Bridge for LEZ

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

Build a trustless, privacy-preserving lock-and-mint bridge that lets ERC-20
tokens (and native ETH, wrapped as WETH) held on Ethereum enter LEZ as canonical
wrapped assets, and exit back to Ethereum on redemption.

An Ethereum-side vault contract escrows deposits. A LEZ-side program mints the
corresponding wrapped token once it has cryptographically verified — with no
trusted intermediary — that the deposit really happened on Ethereum. Redemption
reverses the flow: burning the wrapped token on LEZ entitles the holder to
release the original ERC-20 from the vault, again on cryptographic proof alone.
Verification rests on RISC0 zero-knowledge proofs of consensus and state,
verified in-program on LEZ and via a native precompile on Ethereum.

Crucially, the bridge must achieve this **without letting a public observer
connect a specific Ethereum deposit to the LEZ mint it funded, or a specific LEZ
burn to the Ethereum release it triggered.** That privacy property is a hard
requirement of the same standing as solvency, and it shapes the whole design:
the deposit cannot name its LEZ destination, the burn cannot name its Ethereum
destination, and no component other than the user may hold the information that
connects them.

Teams will need experience with zero-knowledge proof systems, privacy-preserving
protocol design, Solidity smart-contract development, and LEZ program
development.

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
smart-contract expressiveness, so a trustless swap protocol is the only
construction available for them. Ethereum's programmability makes a lock-and-mint
bridge with a vault contract and zero-knowledge proof verification possible
instead.

### A transparent bridge would deanonymise the whole chain

Privacy is not a nice-to-have here; it is the difference between this bridge
strengthening LEZ's privacy guarantees and silently destroying them.

A conventional lock-and-mint bridge publishes a deposit event naming both the
Ethereum depositor and the LEZ recipient. That single event permanently binds a
traceable Ethereum address to a LEZ account, for anyone to read. Because
virtually all external collateral would enter LEZ through this one primitive,
such a bridge becomes the canonical deanonymisation oracle for the entire chain:
an observer needs only to scrape one Ethereum contract's logs to build an
identity map covering most of LEZ's collateral base. Every downstream privacy
feature — private accounts, the privacy-preserving DEX
([RFP-004](./RFP-004-privacy-preserving-dex.md)), shielded lending positions — is
undermined at the point of entry, no matter how well those components protect
data internally. Privacy that leaks at the on-ramp is not privacy.

The redemption leg is symmetric and, if anything, worse: a burn naming its
Ethereum destination publishes the exit address alongside the LEZ account that
funded it, closing the loop and connecting a user's entire LEZ activity to their
Ethereum identity at both ends.

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
posture — cryptographic verification eliminating the trust-in-signers vector
entirely, combined with per-token and global caps and an admin-governed freeze
authority — is designed directly against that track record.

## 🏗 Design Rationale

### The privacy requirement, stated precisely

Two properties must hold against an adversary who observes **all** public state
on both chains, indefinitely, and who may themselves deposit and redeem:

- **P1 (inbound).** Given an Ethereum deposit, the adversary cannot determine
  which LEZ mint it funded, with probability better than uniform over the
  inbound anonymity set.
- **P2 (outbound).** Given a LEZ burn, the adversary cannot determine which
  Ethereum release it triggered, with probability better than uniform over the
  outbound anonymity set.

Both properties are stated relative to an *anonymity set*, and both degrade to
nothing when that set is small. Sizing, measuring and surfacing the anonymity
set is therefore a first-class requirement, not an implementation detail.

### What cannot be hidden

Two facts are fixed by the environment and cannot be designed away. Proposals
must not claim otherwise:

1. **The Ethereum deposit amount is public**, as is the depositor's address —
   it is an ordinary ERC-20 transfer into the vault, sent by the depositor.
2. **The Ethereum release amount and recipient are public** — the vault must
   move real tokens to a real address.

Privacy is therefore preserved by making these public facts *uninformative about
which counterparty they pair with*, not by attempting to hide them. Three
consequences follow, and proposals must address each:

- **Amounts correlate.** A deposit of 1,337.42 USDC followed by a mint of
  1,337.42 wUSDC is matched by elimination regardless of what cryptography sits
  between them. The amount visible on the Ethereum side must not act as a
  fingerprint.
- **Timing correlates.** A deposit followed promptly by a mint, at a quiet
  moment, is matched by inspection. The protocol must not force users into
  correlated timing.
- **Fee payers correlate.** If the user pays their own gas to mint or release,
  the funding source of that account re-identifies them and the construction
  collapses.

### Suggested approach

The following is offered as orientation, not prescription. Proposals are free to
achieve the required properties by other means, provided they argue the case.

A commitment-and-nullifier shielded pool on each leg is the well-understood
construction for this problem: the deposit publishes a commitment rather than a
destination, and the claimant later proves entitlement without revealing which
deposit they are claiming, with a nullifier preventing double-claims. The
redemption leg works the same way in reverse, which requires splitting redemption
into two separately-timed stages so that the burn need not name its Ethereum
destination.

Minting into **private LEZ state** is the natural fit for the destination side,
and keeps the recipient and balance off public view without additional
machinery.

For the amount-correlation problem, restricting transfers to a fixed set of
per-token denominations is the simpler and better-understood option, and is the
expected baseline for this RFP; hiding amounts outright via value commitments is
stronger but considerably heavier, and is specified as a soft requirement below.

For the fee-payer problem, permissionless relayers paid out of the bridged amount
are the conventional answer, with the destination bound into the proof so a
relayer cannot redirect funds or overcharge.

For the trust problem, note that a proof of source-chain consensus and state
contains no user-specific data, so it can be produced by anyone and anchored
on-chain permissionlessly, leaving the user's own device to prove only their
entitlement against it. This keeps user-specific proving cheap enough to run
locally, which matters because whoever generates the entitlement proof learns
the connection the design exists to protect.

### Trust model

This bridge is **trustless**. It requires no trust in signers, validators,
attestor federations, relayers, or proof-generation services. A user relies only
on the correctness of the proof system, the security of the Ethereum network,
and the security of the LEZ network.

Any off-chain participant in the design may be trusted for **liveness only** —
able to decline service, but never able to steal, redirect, forge, censor
selectively, or deanonymise. Proposals must identify every such participant and
justify that each is liveness-only.

Caps and the freeze authority remain in place as operational safety mechanisms
independent of the cryptographic trust model. Note that the freeze authority can
necessarily operate at token and protocol granularity only: it cannot freeze an
individual user's holdings, because the protocol does not know whose are whose.
This is a designed consequence of the privacy guarantees, not a gap, and must be
documented as such.

### Loss of access

In any design meeting these requirements, the protocol cannot identify who owns
what, so there is no administrative recovery path when a user loses access. This
is the largest UX risk in the RFP and must be designed against rather than
disclaimed: a user must be able to recover their full bridge position from
credentials they already hold and already back up. Requiring users to separately
back up new secrets generated during a deposit is not acceptable.

### Token registry and decimal normalisation

Each supported ERC-20 is registered individually by the admin authority: its
Ethereum contract address, its LEZ wrapped-token mint, its decimals, its
permitted transfer amounts, and its per-token caps. ERC-20 tokens do not share a
common decimals convention (6 for USDC, 8 for WBTC, 18 for WETH and DAI); the
wrapped LEZ mint for each token must document its own decimals and the exact
conversion applied on mint and burn, and registration must reject any token
whose values cannot be represented exactly in the chosen LEZ mint precision.

Fee-on-transfer and rebasing ERC-20s break the invariant that the amount
deposited equals the amount mintable. The registry must reject them, and the
vault must verify the actual balance delta on deposit rather than trusting the
requested amount.

### Finality and reorg protection

Wrapped tokens must not be minted against an Ethereum deposit that a reorg could
still remove, and the vault must not release against a LEZ burn that is not yet
final. Both directions require a configured finality condition on the source
chain. The chosen depth is a direct trade-off between user-facing latency and
reorg risk; proposals must document the depth chosen and the residual risk it
leaves.

Once a user's claim is valid it must remain valid indefinitely, since users are
expected to delay their own submissions for privacy reasons and must never be
forced to act promptly to avoid expiry.

### Fee structure

This RFP does not mandate a specific protocol fee rate. Proposals must specify
who pays, when fees are collected, the exact rate, and where fees are routed. A
governance-activatable fee switch with an initial zero rate, gated by the admin
authority per RFP-001, is the recommended baseline, consistent with the pattern
used elsewhere in the Logos RFP set (see
[RFP-017](./RFP-017-token-vesting.md), "Fee structure"). Any fee paid by a user —
protocol or relayer — must take a value that does not distinguish their
transaction from others, since a distinctive fee is itself a fingerprint.

## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

1. Implement an Ethereum vault contract (Solidity) that escrows deposits of any
   ERC-20 in the supported-token registry, plus native ETH (auto-wrapped to
   WETH, ie sends ETH to the canonical WETH contract to immediately receive WETH). The vault must verify the actual balance delta received and reject any
   deposit that does not deliver the expected amount.
2. A deposit must not publish, store, or otherwise reveal its LEZ destination.
   No Ethereum transaction argument, event, or contract state may identify the
   account that will receive the wrapped tokens.
3. Implement a LEZ bridge program that mints the corresponding wrapped token on
   cryptographic verification of a valid Ethereum deposit, using RISC0 proofs of
   Ethereum consensus and state verified in-program. Verification must require
   no trusted party.
4. Wrapped tokens are minted into private LEZ state by default, so that neither
   the recipient nor the balance is publicly visible.
5. Each deposit may be claimed at most once. A repeat claim must be rejected
   deterministically, without minting, and without revealing which deposit it
   referred to.
6. Implement a burn path on the LEZ bridge program that entitles the holder to
   release the original asset from the Ethereum vault. The burn must not
   publish, store, or otherwise reveal its Ethereum destination.
7. The Ethereum vault releases the original ERC-20 (or unwrapped ETH, for WETH
   redemptions) on cryptographic verification of a valid LEZ burn, using a RISC0
   proof verified natively via a precompile (Groth16 verifier or equivalent).
   Each burn may be redeemed at most once.
8. The amounts visible on Ethereum must not identify which mint or burn they
   correspond to. Proposals must state the mechanism chosen (fixed
   denominations are the expected baseline) and its effect on anonymity-set
   size.
9. A user must be able to complete both flows without holding a funded account
   on either chain, and whoever submits or pays for a transaction on the user's
   behalf must not thereby learn, or be able to prove, which deposit or burn it
   corresponds to. That party must not be able to alter the destination or take
   more than an agreed fee.
10. A user must be able to recover their full bridge position — every claimable
    deposit and every unredeemed burn — from credentials they already hold, with
    no dependence on any server-side index and no separately-backed-up secret
    generated during the flow.
11. An admin authority (per RFP-001, integrated via the SPEL framework where
    applicable to the LEZ side) can register a supported ERC-20 (Ethereum
    address, LEZ wrapped mint, decimals, permitted amounts, caps) and deregister
    a token. Registration changes must be mirrored consistently on both sides;
    document how the two stay in sync and what happens if they temporarily
    diverge. The registry must reject fee-on-transfer and rebasing tokens.
12. Global and per-token deposit and redemption caps, configurable by the admin
    authority, bound the maximum value that can be minted or released within a
    rolling window, as a rate limiter independent of the freeze authority. Cap
    enforcement must not require identifying individual users.
13. A freeze authority (per RFP-002) can pause minting and/or redemption, either
    globally or for a single registered token, on the Ethereum vault and the LEZ
    bridge program independently.

#### Usability

1. Provide an SDK that can be used to build Logos modules for: depositing,
   claiming a deposit on LEZ, redeeming, releasing on Ethereum, recovering a
   position from user credentials, and reading the supported-token registry and
   its permitted amounts.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo. It must cover the
   deposit and redemption flows end to end, position recovery, and a registry
   view showing supported tokens, permitted amounts, caps and current
   utilisation.
3. Provide a CLI that covers core functionality: deposit, claim, redeem,
   release, recover, and query the registry. The CLI may have fewer features
   than the mini-app but must support all essential operations.
4. Any long-running off-chain component the design requires must be provided as
   a **Logos module accompanied by a Logos Core headless CLI/daemon**, runnable
   standalone, supporting configurable RPC endpoints for both chains,
   configurable finality depth, structured logging, and a clean shutdown path.
   Proposals must integrate mature RISC0 implementations (e.g. Zisk) rather than
   reimplementing zero-knowledge primitives. Document the operator journey
   end-to-end: install, configure, run, monitor.
5. Provide an IDL for the LEZ bridge program using the
   [SPEL framework](https://github.com/logos-co/spel).
6. Before any privacy-sensitive submission, the mini-app and CLI must show the
   user the current anonymity set their transaction would join, and warn when it
   falls below a configurable threshold. Where delaying a submission improves
   privacy, the client must default to doing so rather than submitting
   immediately, with the user able to inspect and override.
7. Documentation and UI must clearly explain what is public and what is private
   at each step on both chains, in the manner of
   [RFP-004](./RFP-004-privacy-preserving-dex.md), so users can judge their own
   exposure.
8. Return clear, actionable error messages for all failure modes: unsupported
   token, invalid amount, cap exceeded, verification failure, insufficient
   finality, already claimed, and program or per-token frozen. Error messages
   must not reveal which deposit or burn a failed attempt referred to.

#### Reliability

1. Minting is atomic: a failed or rejected claim leaves the deposit claimable on
   retry and consumes nothing.
2. Redemption is atomic at each stage: a failed burn does not destroy wrapped
   tokens without preserving the holder's entitlement to release, and a failed
   release leaves that entitlement intact.
3. No deposit can be claimed twice and no burn redeemed twice, deterministically
   and under adversarial retry.
4. A valid claim remains valid indefinitely; later chain activity must never
   invalidate a user's outstanding entitlement.
5. Position recovery is complete: a client restored from user credentials alone
   must rediscover every claimable deposit and unredeemed burn, verified by a
   test that wipes all local state.
6. Temporary RPC or connectivity failure on either chain leaves any off-chain
   component in a recoverable state, able to resume without duplicating work
   already done.
7. An interrupted user-side operation does not consume, corrupt, or expose the
   user's entitlement.

#### Performance

1. Verifying an inbound claim and minting must complete within a single LEZ
   public transaction at the per-transaction compute budget in force at delivery
   time. Document the compute-unit cost with a breakdown by component, extending
   the measurement methodology from
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md).
2. Document the compute-unit cost of any additional LEZ-side verification the
   design requires beyond the per-claim path, and its amortised per-deposit cost
   at the recommended operating cadence.
3. Document the Ethereum-side gas cost of a deposit, of a release, and of any
   additional on-chain verification the design requires.
4. Any proving the user's own device must perform has to be practical on the
   desktop hardware Basecamp runs on. Measure and document wall-clock time and
   peak memory on a mid-range laptop and on the lowest specification the team
   declares as supported, and state that minimum explicitly.
5. Document end-to-end deposit latency (deposit to LEZ mint) and redemption
   latency (burn to Ethereum release), each broken down by source-chain finality
   wait, proof generation, any privacy-motivated delay, and on-chain
   verification.
6. Document the compute resources (CPU, RAM, time) required to run any
   off-chain component the design requires.
7. Document the growth rate and on-chain storage cost of all bridge state that
   accumulates with usage, with projections at 1M and 10M operations.

#### Supportability

1. The Ethereum vault contract and the LEZ bridge program are deployed and
   tested on a public Ethereum testnet and LEZ devnet/testnet respectively.
2. End-to-end integration tests exercise the full deposit and redemption round
   trip against a LEZ sequencer (standalone mode) and an Ethereum test network
   or local fork, and are included in CI. CI must be green on the default
   branch.
3. Every hard requirement in Functionality, Usability, Reliability, Performance,
   and Privacy Preservation has at least one corresponding test.
4. A README documents end-to-end usage: contract and program addresses,
   deployment steps for both chains, and step-by-step instructions for
   depositing and redeeming via CLI and mini-app.
5. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for both flows
   including position recovery.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI and any operator-facing components, covering the core user and
   operator journeys respectively.
7. Provide Figma designs or equivalent for all mini-app GUI artefacts, including
   the anonymity-set disclosure and the recovery flow.
8. The Ethereum vault contract undergoes an independent third-party
   smart-contract security audit before mainnet deployment; the audit report (or
   a summary, if the full report is not publishable) must be linked from the
   README. This requirement exists because cross-chain bridges are the single
   most-attacked category of DeFi infrastructure (Chainalysis has tracked more
   than $2.8B stolen from bridges since 2022); it is not optional.
9. Provide a **privacy properties document** covering: a formal statement of P1
   and P2 and the anonymity set each is measured against; exactly what is
   visible on-chain at every step on both chains; what an adversary observing
   all public state can and cannot infer; what every off-chain participant in
   the design can observe; residual leakage from timing, amount selection, fee
   payment, network metadata and usage patterns; and the conditions under which
   the guarantees degrade or fail.
10. Document the anonymity-set growth model: expected set size over time at
    projected volumes, the minimum below which the guarantees are considered not
    to hold, and guidance for users bridging before the pool has matured.

#### + Bridge Security

1. Proof verification must be deterministic and independently verifiable. Both
   the LEZ program and the Ethereum vault must reject invalid proofs, tested
   with incorrect public inputs, proofs for incorrect chain state, tampered
   headers, and replayed proofs.
2. Any off-chain component must independently derive chain state from the source
   chain rather than accepting data supplied by a third party without
   verification.
3. A malicious party submitting on a user's behalf must not be able to redirect
   funds, inflate their fee, or replay the user's submission to a different
   destination. Test each case explicitly.
4. Caps (Functionality #12) bound the maximum value at risk in any rolling
   window; proposals must document recommended defaults and the reasoning behind
   them.
5. The freeze authority (Functionality #13) must be exercisable independently on
   each half, so either can be paused without the other being operational or
   reachable.
6. Soundness of supply: total wrapped supply on LEZ must never exceed the
   vault's holdings. Provide tests attempting to mint without a valid deposit,
   mint twice from one deposit, and release without a valid burn.
7. User-facing documentation must state the trustless verification model and the
   liveness-only role of any off-chain participant (see Design Rationale, "Trust
   model").

#### + Privacy Preservation

1. **P1 must hold under test.** Provide an automated test that constructs a
   population of deposits and mints and asserts that no correlation derivable
   from public state identifies the true pairing better than chance across the
   anonymity set.
2. **P2 must hold under test.** The equivalent test for burn-to-release
   pairings.
3. No transaction argument, event, log, or account-state change on either chain
   may reveal a deposit's LEZ destination or a burn's Ethereum destination.
   Provide a test asserting this over full event and state diffs for a complete
   round trip.
4. Information that would connect the two legs must never leave the user's
   control. Document every component that handles user data, and provide a test
   asserting such information is absent from all submitted transaction data.
5. Failure and error paths must not reveal which deposit or burn was involved: a
   rejected claim, a repeat claim, and a cap rejection must be
   indistinguishable in that respect.
6. The client must not make any network request that reveals which deposit or
   burn it is acting on. Document every network call made during a
   privacy-sensitive operation and justify each.
7. Where a third party submits on the user's behalf, document precisely what
   that party learns, and ensure the user can switch between such parties per
   operation.
8. The default configuration must be the private one. No user action may be
   required to obtain the privacy guarantees, and any override that weakens them
   must require explicit confirmation.

### Soft Requirements

1. **Hidden amounts.** Remove the amount-visibility constraint entirely by
   concealing transferred values on the LEZ side, rather than relying on a fixed
   set of permitted amounts. This merges all per-amount anonymity sets into one
   and removes the need for users to split transfers. Whatever is delivered
   under the hard requirements should be designed so this can be adopted later
   without redeploying the vault or resetting accumulated anonymity; document the
   intended migration path even if it is not implemented.
2. Batching: amortise verification cost across multiple operations in a single
   transaction, analogous to the multi-feed batching soft requirement in
   RFP-020. Batching also improves privacy by making individual operations
   harder to isolate.
3. Optional viewing keys allowing a user to *voluntarily* disclose their own
   bridge activity to a chosen third party, without weakening privacy for anyone
   else and without any protocol-level disclosure capability.
4. A configurable per-token release delay, in addition to finality and any
   user-chosen delay, as an extra circuit-breaker window allowing the freeze
   authority to react to anomalous redemption volume before funds leave the
   vault.
5. Support for wrapping ERC-20 tokens from additional EVM chains (e.g. Arbitrum,
   Base) behind the same LEZ bridge program, reusing the registry and caps.
   Consider whether anonymity sets should be shared across source chains to
   enlarge them.
6. Hardware acceleration as an optional path for users with capable machines,
   without making it a requirement.
7. Design the proof-system components as pluggable, so that future zkVM
   improvements, proof compression, or hardware acceleration can be adopted
   without restructuring the vault or the bridge program.

### Out of Scope

The following are explicitly excluded from this RFP:

- Wrapping non-fungible assets (ERC-721, ERC-1155).
- Bitcoin, Monero, and Zcash bridging: these have a dedicated trustless path via
  atomic swaps, delivered in [RFP-003](./RFP-003-atomic-swaps.md).
- Mobile and in-browser proving. Basecamp is a desktop application and is the
  delivery surface for the mini-app, so any user-side proving targets desktop
  only. Proposals must not constrain their design to fit mobile-class resource
  budgets.
- Network-level anonymity. The guarantees here are properties of on-chain state.
  IP-level correlation between a user's Ethereum deposit and their later LEZ
  activity is out of scope as an implementation concern, but must be disclosed
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
deregisters supported tokens and configures permitted amounts and caps. These
admin-gated functions require the standardised admin authority library from
[RFP-001](./RFP-001-admin-authority-lib.md).

#### Freeze authority (RFP-002)

The Bridge Security requirements specify a freeze authority able to pause
minting and/or redemption, globally or per token, as a circuit breaker
independent of the caps. This requires the standardised freeze authority library
from [RFP-002](./RFP-002-freeze-authority-lib.md).

#### Token mint/burn authorities (LP-0013)

The LEZ bridge program mints wrapped tokens on verified deposits and burns them
on redemption. This requires the token mint/burn authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md).
Note that Soft Requirement #1 (hidden amounts) would require supply accounting
over concealed values, which is likely an extension beyond what LP-0013
provides; proposals pursuing that path must state what extension is needed.

#### Private LEZ account state

Wrapped tokens are minted into private LEZ state, and redemptions spend from it.
The privacy guarantees depend on this: minting into a public account exposes the
recipient immediately. Proposals must state which LEZ private-state primitives
they rely on and their maturity.

#### RISC0 zkVM

The bridge verifies proofs of consensus and state in-program on LEZ and natively
on Ethereum. This requires RISC0, a production-ready zkVM. Proposals must
leverage mature RISC0 implementations (e.g.
[Zisk](https://github.com/risc0/zisk)) rather than building custom circuits.

### Soft dependencies

#### Event emission (LP-0012)

Structured on-chain events allow clients and off-chain components to react
without polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
is **closed** (delivered).

## 👤 Recommended Team Profile

Team experienced with:

- Zero-knowledge proof systems and RISC0 zkVM (guest program development, proof
  generation and verification, public/private input handling)
- Privacy-preserving protocol design, including anonymity-set analysis and the
  ability to reason rigorously about what public state does and does not reveal
- Solidity smart-contract development, including experience preparing a contract
  for third-party security audit, and EVM precompile integration
- Cryptographic primitives (Groth16 verification, Merkle proofs, consensus
  verification, state root inclusion proofs)
- LEZ program development, private-state programs, and on-chain proof
  verification
- Smart-contract security auditing (proof validation, replay attacks, reorg
  handling, cap bypass, privacy-leak analysis)
- Cross-chain system design and integration testing (dual-chain atomic
  operations, finality assumptions, determinism and reproducibility)

## ⏱ Timeline Expectations

Estimated software delivery duration: **16–20 weeks**. This is longer than a
transparent lock-and-mint bridge would require; the privacy construction,
user-side proving, position recovery, and the privacy test suite are the
additional scope. This excludes the third-party audit lead time required before
mainnet deployment (Supportability #8), which is typically procured and
scheduled separately.

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
