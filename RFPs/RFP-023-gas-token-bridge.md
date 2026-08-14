---
id: RFP-023
title: Native Gas Token Bridge for LEZ
tier: M
status: open
category: Developer Tooling & Infrastructure
dependencies:
  - id: RFP-021
    reason: Delivers the vault, mint, registry, caps and privacy construction this RFP inverts; the two bridges share their patterns and their proof-verification approach.
  - id: RFP-022
    reason: The trustless Ethereum state attestation primitive is what the LEZ vault program uses to verify an ERC-20 burn really happened on Ethereum, as specified in Functionality.
  - id: RFP-001
    reason: Admin authority governs the caps, the finality depth, and the chain and zone pairing, as specified in Functionality.
  - id: RFP-002
    reason: Freeze authority provides the circuit breaker to halt minting and/or release if a proof-system or vault vulnerability is suspected.
  - id: LP-0013
    reason: Token authority primitives are required for the LEZ vault program to escrow and release native gas token under program control.
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-023 — Native Gas Token Bridge for LEZ

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

Build the reverse-direction counterpart to
[RFP-021](./RFP-021-wrapped-erc20.md): a trustless, privacy-preserving bridge in
which **LEZ is the vault** and Ethereum is the minter, so a zone's native gas
token can be represented as an ERC-20 on Ethereum and acquired by someone who
holds nothing on the zone yet.

A LEZ-side vault program escrows native gas token. Locking gas token in that
vault entitles the holder to mint the corresponding ERC-20 on Ethereum, on
cryptographic proof of the lock. Burning that ERC-20 on Ethereum entitles the
holder to release native gas token from the LEZ vault, on cryptographic proof of
the burn, verified in-program on LEZ via the attestation primitive from
[RFP-022](./RFP-022-ethereum-state-attestation.md).

This is RFP-021's construction with the chains swapped, and it inherits that
RFP's requirements accordingly: the same trustless verification model, the same
unlinkability guarantees, the same caps and freeze authority, the same immutable
verifier preference. It is a separate deliverable because the roles invert. The
vault logic and the mint logic move to opposite chains, which makes the LEZ
program the thing holding value and the Ethereum contract the thing issuing a
representation, with a correspondingly different audit target.

One problem is genuinely new and has no analogue in RFP-021: releasing native
gas token to a recipient who, by construction, has no gas with which to pay for
the release.

Teams will need experience with zero-knowledge proof systems, privacy-preserving
protocol design, Solidity smart-contract development, and LEZ program
development.

## 🔥 Why This Matters

Every other LEZ application RFP assumes the user already holds gas. The
privacy-preserving DEX ([RFP-004](./RFP-004-privacy-preserving-dex.md)), the
lending protocol ([RFP-008](./RFP-008-lending-borrowing-protocol.md)), the
curated vaults ([RFP-012](./RFP-012-curated-lending-vaults.md)), the reflexive
stablecoin ([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), the
liquidation engine ([RFP-014](./RFP-014-liquidation-auction-engine.md)), both
launchpads ([RFP-015](./RFP-015-bonding-curve-launchpad.md),
[RFP-016](./RFP-016-lbp-launchpad.md)) and token vesting
([RFP-017](./RFP-017-token-vesting.md)) all converge on the same pattern: gas is
funded by an atomic deshield from the user's own shielded balance, in a single
indivisible action, precisely so that no external funding source links the
ephemeral account to an existing identity. RFP-004 states the constraint
directly: funding the operation account from any external source, such as a CEX
withdrawal or a known wallet, creates an on-chain link and breaks the privacy
guarantee.

That pattern is correct, and it presupposes the user already has funds on LEZ.
It says nothing about how they got there. A user arriving with nothing cannot
deshield, cannot pay for a first transaction, and cannot use any of the above.
The conventional answers, a faucet or a centralised on-ramp, are either unsuited
to production or reintroduce exactly the identity link the atomic-deshield
pattern exists to avoid.

This RFP is the trustless entry path. Someone who holds ETH and an Ethereum
wallet can acquire native gas token by burning an ERC-20 representation, with no
custodian, no faucet operator, and no funding source that links their new LEZ
account to their Ethereum identity. It closes the loop that RFP-021 opens:
RFP-021 brings external *value* onto LEZ as wrapped assets, but a user still
needs gas to do anything with it, and wrapped USDC does not pay for a
transaction. Together the two bridges make a user's first interaction with a
zone possible without trusting anyone.

The same mechanism serves an operational need beyond onboarding. A zone's native
gas token becomes transferable and tradeable on Ethereum, where liquidity and
tooling already exist, which gives the token a market that does not depend on
the zone having bootstrapped its own DEX first.

## 🏗 Design Rationale

### Inverted roles

In RFP-021, the Ethereum contract is the vault and the LEZ program is the
minter. Here the roles swap:

- **Outbound (LEZ to Ethereum).** A user locks native gas token in the LEZ vault
  program. Proof of that lock entitles the holder to mint the corresponding
  ERC-20 on Ethereum. The proof is verified natively on Ethereum, the same
  problem RFP-021 solves for its burn-to-release leg.
- **Inbound (Ethereum to LEZ).** A user burns the ERC-20 on Ethereum. Proof of
  that burn entitles the holder to release native gas token from the LEZ vault.
  The proof is verified in-program on LEZ, consuming the attestation primitive
  from [RFP-022](./RFP-022-ethereum-state-attestation.md) rather than rebuilding
  Ethereum consensus and inclusion verification.

Everything else follows RFP-021. The requirements below mirror it item for item
wherever the mirroring is exact, and say so rather than restating the reasoning.

### Supply is demand-driven, and the ERC-20 has no privileged minter

The ERC-20 is not a float that anyone stocks in advance. It comes into existence
only when someone locks native gas token in the LEZ vault, and it leaves
existence only when someone burns it to release that gas token. Supply tracks
demand by construction, exactly as the wrapped-token supply does in RFP-021, and
no seeding, replenishment, or market-making mechanism is required to make the
mechanism work.

The invariant that makes this sound is that the ERC-20 has **no privileged
minter and no pre-issued supply**. Every unit in existence is backed by gas
token locked on LEZ; burning is the only way to unlock it. An admin authority, a
deployer, or any other party must have no path to mint a single unit without a
corresponding lock. This is stated as a testable requirement in Bridge Security
#5.

That invariant is not a theoretical concern. Meter Passport's auto-wrap and
unwrap convenience feature for native gas tokens did not properly restrict
direct interaction with the wrapped ERC-20 contracts, nor verify that a matching
real value transfer had occurred, which let an attacker mint WETH and WBNB
without depositing collateral, for a direct loss of roughly $4.3M and a further
~$3.3M cascading into a dependent lending protocol (see
[Appendix: Bridges and Wrapped Tokens](../appendix/bridges-and-wrapped-tokens.md)).
It is the one documented cross-chain bridge hack specifically about wrapping
native gas tokens, and it is the exact failure this RFP's invariant and its
associated tests exist to foreclose.

One consequence is worth documenting for users rather than designing against: a
newcomer can only obtain the ERC-20 if someone has already moved gas token
outward to Ethereum and made it available. That is an ordinary liquidity
property of any two-way bridge, not a protocol requirement, and the
documentation must set the expectation plainly rather than implying the bridge
manufactures gas from nothing.

### The gas circularity

Releasing native gas token from the LEZ vault requires a LEZ transaction, and
the recipient of that release is, in the motivating case, someone who holds no
gas token at all. If claiming the release requires the claimant to already hold
gas, the bridge does not solve the problem it exists to solve.

This RFP requires the release path to work for a recipient with a zero balance
(Functionality #4) but does not mandate how. Several shapes are viable, and
which is best depends on LEZ capabilities at delivery time: a permissionless
relayer paid out of the released amount, with the destination bound into the
proof so the relayer can neither redirect funds nor overcharge, which is the
mechanism RFP-021 already relies on for its fee-payer mitigation; a
protocol-level fee abstraction if LEZ offers one; or sequencer-level sponsorship
for this specific program.

Whatever the mechanism, it is bound by RFP-021's trust model: no specific
relayer or off-chain party may be a required counterparty, any such role must be
permissionless so that a single participant declining to act never blocks the
user, and no participant may be able to steal, redirect, forge, censor
selectively, or deanonymise. It is also bound by the privacy requirement below,
which is the harder constraint: whoever pays for the release must not become a
correlation signal that links the Ethereum burn to the LEZ account receiving the
gas.

### Privacy mirrors RFP-021 exactly

The privacy construction carries over with the chains swapped, and it carries
over in full. Each leg of each bridge has exactly one public endpoint, and it is
the Ethereum endpoint in both:

- RFP-021 inbound: public Ethereum deposit, hidden LEZ mint destination.
- RFP-023 inbound: public Ethereum burn, hidden LEZ release destination.
- RFP-021 outbound: hidden LEZ burn, public Ethereum release recipient.
- RFP-023 outbound: hidden LEZ lock, public Ethereum mint recipient.

Minting an ERC-20 to a public Ethereum address reveals neither more nor less
than releasing one from escrow to a public Ethereum address. The hard
requirement is therefore identical to RFP-021's, with the terms substituted:
**no information other than amount, token, and timing may enable an adversary
who observes all public state on both chains, indefinitely, and who may
themselves use the bridge, to link an Ethereum burn to the LEZ release it
funded, or a LEZ lock to the Ethereum mint it triggered.**

The three correlation points from RFP-021, "The privacy requirement, stated
precisely," apply unchanged and proposals must address each: amounts correlate
unless transfers are restricted to fixed per-token denominations; timing
correlates unless the protocol permits and the UI encourages delay; and fee
payers correlate, which here is sharpened by the gas circularity, since the
party paying for a release is structurally more visible than a user paying their
own way.

### Finality, caps, freeze, and pairing

These follow RFP-021 without modification in substance: a configured finality
condition on the source chain in each direction, admin-configurable per
deployment; global and per-deployment caps as a rate limiter independent of the
freeze authority; a freeze authority exercisable independently on each half; and
a strict pairing between a specific LEZ zone and program and a specific Ethereum
contract and chain ID, so a proof valid for one pairing is never accepted for
another. A claim once valid remains valid indefinitely, since users are expected
to delay their own submissions for privacy reasons.

## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

01. Implement a LEZ vault program that escrows native gas token. Locking must
    support the gas token being held in, and locked from, either a private or a
    public LEZ account, at the holder's choice.
02. A lock must not publish, store, or otherwise reveal its Ethereum
    destination. No LEZ transaction argument, event, or account-state change may
    identify the address that will receive the minted ERC-20.
03. Implement an Ethereum contract that mints the ERC-20 representation on
    cryptographic verification of a valid LEZ lock, verified natively on
    Ethereum. Verification must require no trusted party.
04. The LEZ vault releases native gas token on cryptographic verification of a
    valid ERC-20 burn on Ethereum, consuming the attestation primitive from
    [RFP-022](./RFP-022-ethereum-state-attestation.md) rather than implementing
    its own Ethereum consensus and inclusion verification. **Claiming that
    release must not require the recipient to already hold native gas token**,
    and no specific relayer or other off-chain party may be a required
    counterparty (see Design Rationale, "The gas circularity"). Test the release
    path end to end with a recipient account holding a zero balance.
05. A burn on Ethereum must not publish, store, or otherwise reveal its LEZ
    destination.
06. Releasing must support both a private LEZ account and a public LEZ account
    as the destination, at the burner's choice.
07. Uniqueness is enforced in both directions: no lock can be minted against
    twice and no burn released against twice, deterministically and under
    adversarial retry. On the LEZ side this is keyed on the statement identifier
    the RFP-022 attestation carries.
08. The amounts visible on Ethereum must not identify which lock or release they
    correspond to. Proposals must state the mechanism chosen (fixed
    denominations are the expected baseline) and its effect on anonymity-set
    size.
09. A user must be able to recover every one of their own unclaimed locks and
    unreleased burns from credentials they already hold, with no dependence on
    any server-side index and no separately-backed-up secret generated during
    the flow, on the same terms as RFP-021, "Loss of access."
10. An admin authority (per [RFP-001](./RFP-001-admin-authority-lib.md),
    integrated via the [SPEL framework](https://github.com/logos-co/spel) where
    applicable to the LEZ side) can configure the caps, the finality depth, and
    the fee parameters per deployment.
11. Global caps, configurable by the admin authority, bound the maximum value
    that can be minted or released within a rolling window, as a rate limiter
    independent of the freeze authority. Cap enforcement must not require
    identifying individual users.
12. The finality depth required before a lock may be minted against, and before
    a burn may be released against, is configurable by the admin authority per
    deployment. A change to the configured depth must not invalidate a claim
    that was already valid under the previous depth.
13. A freeze authority (per [RFP-002](./RFP-002-freeze-authority-lib.md)) can
    pause minting and/or release, on the Ethereum contract and the LEZ vault
    program independently.
14. Each LEZ vault program deployment refers to a specific Ethereum contract
    deployment on a specific chain (contract address plus chain ID), and
    reciprocally each Ethereum contract deployment refers to a specific zone
    instance (LEZ blockchain ID, zone ID, and program ID), with each pairing
    checked as part of proof verification so a lock or burn valid for one
    pairing is never accepted as valid for another. The same program and
    contract design must be deployable, unmodified, against any EVM chain,
    mainnet or testnet.
15. The design must let multiple entities each operate under their own
    independent configuration (caps, fees, admin authority, finality depth), on
    the same or different pairs of blockchain programs, with strict separation
    between them: one entity's configuration must have no privileged access over
    another's configuration or funds. Document how a client identifies and
    switches between configurations.
16. A protocol fee may be charged on minting and on release, at a rate
    configurable by the admin authority per deployment, including zero. The fee
    value must not distinguish a user's transaction from others, consistent with
    Functionality #8. Where the release-path mechanism compensates a relayer out
    of the released amount, document how that compensation interacts with the
    fixed-denomination requirement.

#### Usability

1. Build core functionalities for both users and admin in a Logos core module,
   enabling the delivery of different Logos ui modules: locking gas token,
   claiming the Ethereum mint, burning the ERC-20, claiming the LEZ release,
   recovering a position from user credentials, and reading and administering
   the configuration.
2. Provide a Logos mini-app, aka Logos ui module, covering both flows end to
   end, position recovery, and a view showing permitted amounts, caps and
   current utilisation. Also provide a UI for the admin functionality; whether
   this is combined into one UI or delivered as two separate ones is left to the
   applicant's choice.
3. The onboarding flow must be usable by someone who holds nothing on the zone.
   The mini-app must not require a funded LEZ account to complete a release, and
   must not present a step that silently assumes one.
4. Any long-running off-chain component the design requires must be provided as
   a **Logos module accompanied by a Logos Core headless CLI/daemon**, runnable
   standalone, supporting configurable RPC endpoints for both chains,
   configurable finality depth, structured logging, and a clean shutdown path.
   Document the operator journey end-to-end: install, configure, run, monitor.
5. Provide an IDL for the LEZ vault program using the
   [SPEL framework](https://github.com/logos-co/spel).
6. The mitigations to the three correlation points in Design Rationale, "Privacy
   mirrors RFP-021 exactly" (amount, timing, fee payer) must be enabled by
   default. The mini-app and CLI must show a clear indicator of what data would
   be leaked by the user's current choices, and default to the recommended
   parameters rather than requiring the user to select them.
7. The mini-app and CLI must default to inviting the user to release into, and
   lock from, a private account: the private path is the pre-selected option,
   and choosing the public path requires an explicit action, consistent with
   Privacy Preservation #8.
8. Documentation and UI must clearly explain what is public and what is private
   at each step on both chains, and must set the expectation described in Design
   Rationale, "Supply is demand-driven", that the ERC-20 is available only to
   the extent someone has moved gas token outward.
9. Return clear, actionable error messages for all failure modes: invalid
   amount, cap exceeded, verification failure, insufficient finality, already
   claimed, and program frozen. Error messages must not reveal which lock or
   burn a failed attempt referred to.

#### Reliability

1. Minting is atomic: a failed or rejected mint claim leaves the lock claimable
   on retry and consumes nothing.
2. Release is atomic: a failed burn does not destroy the ERC-20 without
   preserving the holder's entitlement to release, and a failed release leaves
   that entitlement intact.
3. No lock can be minted against twice and no burn released against twice,
   deterministically and under adversarial retry.
4. A valid claim remains valid indefinitely; later chain activity must never
   invalidate a user's outstanding entitlement.
5. Position recovery is complete: a client restored from user credentials alone
   must rediscover every claimable lock and unreleased burn, verified by a test
   that wipes all local state.
6. Temporary RPC or connectivity failure on either chain leaves any off-chain
   component in a recoverable state, able to resume without duplicating work
   already done.
7. An interrupted user-side operation does not consume, corrupt, or expose the
   user's entitlement.
8. Proposals must integrate mature, audited proof-system implementations rather
   than reimplementing zero-knowledge primitives from scratch.
9. CI must be green on the default branch.

#### Performance

1. Verifying a burn and releasing gas token must complete within a single LEZ
   transaction at the per-transaction compute budget in force at delivery time.
   Document the compute-unit cost with a breakdown by component, separating the
   RFP-022 attestation verification cost from this RFP's own logic, and
   extending the measurement methodology from
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md).
2. Document the Ethereum-side gas cost of a mint, of a burn, and of the LEZ-lock
   proof verification the mint requires.
3. Any proving the user's own device must perform has to be practical on the
   desktop hardware Basecamp runs on. Measure and document wall-clock time and
   peak memory on a mid-range laptop and on the lowest specification the team
   declares as supported, and state that minimum explicitly.
4. Document end-to-end latency in both directions, each broken down by
   source-chain finality wait, proof generation, any privacy-motivated delay,
   and on-chain verification.
5. Document the compute resources (CPU, RAM, time) required to run any off-chain
   component the design requires.
6. Document the growth rate and on-chain storage cost of all bridge state that
   accumulates with usage, with projections at 1M and 10M operations.

#### Supportability

01. The LEZ vault program and the Ethereum contract are deployed and tested on a
    LEZ testnet and a public Ethereum testnet respectively.
02. End-to-end integration tests exercise the full round trip in both directions
    against a LEZ sequencer (standalone mode) and an Ethereum test network or
    local fork, and are included in CI. One test must cover the motivating
    journey end to end: a user with no LEZ account and no gas token acquires gas
    and completes a first unrelated LEZ transaction.
03. Every hard requirement in Functionality, Usability, Reliability,
    Performance, and Privacy Preservation has at least one corresponding test.
04. A README documents end-to-end usage: contract and program addresses,
    deployment steps for both chains, and step-by-step instructions for both
    directions via CLI and mini-app.
05. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the core module, covering the developer integration journey for both
    flows including position recovery.
06. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the CLI and any operator-facing components, covering the core user and
    operator journeys respectively.
07. Submit a
    [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
    for the deployer journey, covering how an entity stands up its own
    independently configured deployment (Functionality #15).
08. The Ethereum contract undergoes an independent third-party smart-contract
    security audit before mainnet deployment; the audit report must be
    published. The audit scope must explicitly include the mint authorisation
    path, given the Meter Passport precedent in Design Rationale.
09. Provide a **privacy properties document** on the same terms as RFP-021,
    Supportability #9, covering: a formal statement of Privacy Preservation #1
    and #2 and the anonymity set each is measured against; exactly what is
    visible on-chain at every step on both chains; what an adversary observing
    all public state can and cannot infer; what every off-chain participant can
    observe, with specific attention to whoever pays for a release; residual
    leakage from timing, amount selection, fee payment, network metadata and
    usage patterns; and the conditions under which the guarantees degrade.
10. Document the anonymity-set growth model: expected set size over time at
    projected volumes, the minimum below which the guarantees are considered not
    to hold, and guidance for users bridging before the pool has matured.
11. The UI must let users change the targeted Ethereum RPC address and the
    targeted LEZ sequencer or zone.
12. The deliverable must be published on the module catalog.
13. The repository must use the standard Logos GitHub Actions.

#### + Bridge Security

1. Proof verification must be independently verifiable. Both the LEZ vault
   program and the Ethereum contract must reject invalid proofs, tested with
   incorrect public inputs, proofs for incorrect chain state, tampered headers,
   and replayed proofs.
2. A malicious party submitting on a user's behalf must not be able to redirect
   funds, inflate their fee, or replay the user's submission to a different
   destination: a lock and a burn must each secretly commit to the destination
   on the other chain, so that the claim can only be completed by whoever holds
   the seed or credentials that produced the commitment. Test that an adversary
   who observes everything public about a lock or burn, but does not hold the
   originating seed, cannot construct a valid claim for a different destination.
   This applies with particular force to the release path, where a relayer may
   be submitting on behalf of a user who cannot submit for themselves.
3. Caps (Functionality #11) bound the maximum value at risk in any rolling
   window; proposals must document recommended defaults and the reasoning.
4. The freeze authority (Functionality #13) must be exercisable independently on
   each half, so either can be paused without the other being operational or
   reachable.
5. **Soundness of supply.** Total supply of the ERC-20 on Ethereum must never
   exceed the native gas token locked in the LEZ vault. The ERC-20 must have no
   privileged minter and no pre-issued supply: every unit in existence is backed
   by gas token locked on LEZ, and burning it is the only way to unlock that gas
   token. Provide tests attempting to mint without a valid lock, mint twice from
   one lock, release without a valid burn, and release twice from one burn.
   Testing must explicitly include attempts to mint via any admin, deployer,
   owner, or upgrade path, and any path that interacts with the ERC-20 contract
   directly rather than through the bridge entry point, since that is the
   documented Meter Passport failure mode (see Design Rationale, "Supply is
   demand-driven, and the ERC-20 has no privileged minter").
6. User-facing documentation must state the trustless verification model and the
   liveness-only role of any off-chain participant, including whoever submits a
   release on a user's behalf.
7. The verifier (the LEZ vault program and the Ethereum contract's proof
   verification logic) must be deployed as an immutable program with an explicit
   migration path (deploy a new version, drain and redirect to it) in preference
   to an upgradeable contract governed by a mutable key, for the reasons set out
   in RFP-021, Bridge Security #7 and
   [Appendix: Bridges and Wrapped Tokens](../appendix/bridges-and-wrapped-tokens.md).
   Proposals must document the chosen migration mechanism and how in-flight
   locks and burns are honoured across a migration.
8. The freeze authority stops new activity but does not by itself recover funds
   already at risk or resolve locks and burns left in-flight once a
   vulnerability in the verification logic is found. Proposals must specify a
   failsafe strategy, constrained as in RFP-021, Bridge Security #8: any
   recovery must still be claimed by the locker or burner proving their own
   entitlement, not by an admin authority identifying who owns what; it must not
   be able to mint, redirect, or release to any destination other than the one
   the proof specifies; and it must not act on funds beyond what a specific,
   proven vulnerability put at risk. If no mechanism satisfying these
   constraints is achievable, the proposal must document why and what happens to
   affected funds in its absence.

#### + Privacy Preservation

1. **Inbound unlinkability must hold under test.** No signal other than amount
   and timing may narrow down which LEZ release an Ethereum burn funded, beyond
   uniform probability over the remaining candidates. Provide an automated test
   that constructs a population of burns and releases and asserts that no
   correlation derivable from public state, other than amount and timing,
   identifies the true pairing better than chance across the anonymity set those
   signals leave unresolved.
2. **Outbound unlinkability must hold under test.** No signal other than amount
   and timing may narrow down which Ethereum mint a LEZ lock triggered, beyond
   uniform probability over the remaining candidates. The equivalent test for
   lock-to-mint pairings.
3. No transaction argument, event, log, or account-state change on either chain
   may reveal a lock's Ethereum destination or a burn's LEZ destination. Provide
   a test asserting this over full event and state diffs for a complete round
   trip.
4. Information that would connect the two legs must never leave the user's
   control. Document every component that handles user data, and provide a test
   asserting such information is absent from all submitted transaction data.
5. Failure and error paths must not reveal which lock or burn was involved: a
   rejected claim, a repeat claim, and a cap rejection must be indistinguishable
   in that respect.
6. The client must not make any network request that reveals which lock or burn
   it is acting on. Document every network call made during a privacy-sensitive
   operation and justify each.
7. **The release payer must not become a correlation signal.** Whoever pays for
   a release, whether a relayer, a sponsor, or the protocol itself, must not
   thereby link the Ethereum burn to the LEZ account receiving the gas. Document
   precisely what that party learns, ensure the user can switch between such
   parties per operation, and provide a test asserting that the payer's identity
   and payment do not narrow the anonymity set. This is the requirement most at
   risk from the gas circularity and must be treated as a primary design
   constraint, not a late mitigation.
8. The default configuration must be the private one. No user action may be
   required to obtain the privacy guarantees, and any override that weakens them
   must require explicit confirmation.

### Soft Requirements

1. **Hidden amounts.** Remove the amount-visibility constraint entirely by
   concealing transferred values on the LEZ side, rather than relying on a fixed
   set of permitted amounts. Whatever is delivered under the hard requirements
   should be designed so this can be adopted later without redeploying the
   Ethereum contract or resetting accumulated anonymity; document the intended
   migration path even if it is not implemented.

2. **Batching.** Amortise verification cost across multiple operations in a
   single transaction, analogous to the batching soft requirement in
   [RFP-020](./RFP-020-redstone-oracle-adaptor.md).

3. **Shared components with RFP-021.** Where the two bridges genuinely share
   logic (fixed-denomination handling, position recovery, relayer submission,
   the privacy test harness), factor it so both can consume one implementation
   rather than maintaining two divergent copies. Document what is shared and
   what is necessarily distinct.

4. **Optional viewing keys** allowing a user to *voluntarily* disclose their own
   bridge activity to a chosen third party, without weakening privacy for anyone
   else and without any protocol-level disclosure capability.

5. **Additional EVM chains**, each served by its own deployment per
   Functionality #14 (one program per chain ID, not one program juggling several
   chains internally).

6. **Generalisation beyond the gas token.** If the design generalises at no
   material cost to other LEZ-native assets, so that a LEZ-native token other
   than the gas token can be represented on Ethereum by the same mechanism,
   deliver or document that generalisation. This is explicitly not required: the
   gas token is the motivating case and a design specialised to it is
   acceptable.

7. **Pluggable proof components**, so that future zkVM improvements, proof
   compression, or hardware acceleration can be adopted without restructuring
   the vault or the contract.

### Out of Scope

The following are explicitly excluded from this RFP:

- **Wrapping external ERC-20s and ETH into LEZ.** That is the primary flow,
  owned by [RFP-021](./RFP-021-wrapped-erc20.md). This RFP inverts the
  direction, it does not duplicate the primary flow.
- **The Ethereum state attestation primitive.** Verifying Ethereum consensus,
  finality, and state inclusion is owned by
  [RFP-022](./RFP-022-ethereum-state-attestation.md). This RFP consumes that
  primitive, it does not define or rebuild it.
- **Seeding, replenishment, or market-making for the ERC-20.** Supply is
  demand-driven by construction (see Design Rationale); there is no float to
  stock and no incentive mechanism to design.
- **Fiat on-ramps, faucets, and centralised distribution** of the gas token.
  This RFP delivers a trustless path for a user who already holds assets on
  Ethereum; it does not address a user who holds nothing anywhere.
- **Network-level anonymity.** The guarantees here are properties of on-chain
  state. IP-level correlation is out of scope as an implementation concern, but
  must be disclosed as residual leakage under Supportability #9.
- **Protocol-level compliance, disclosure, or selective-deanonymisation
  mechanisms.** Voluntary user-held viewing keys are Soft Requirement #4; any
  capability allowing a third party to deanonymise a user without their consent
  is contrary to the design and out of scope.
- **Circuit optimisation or custom zkVM accelerators for the LEZ side.** LEZ
  runs on RISC0, so proposals should leverage mature existing implementations
  rather than implementing novel circuits.

## ⚠ Platform Dependencies

### Hard dependencies

#### Wrapped ERC-20 and Ether bridge (RFP-021)

This RFP is the reverse-direction counterpart of
[RFP-021](./RFP-021-wrapped-erc20.md) and inherits its vault and mint patterns,
its trust model, its privacy construction, and its approach to caps, freeze
authority, finality and position recovery. RFP-021 establishes those patterns
and carries the reasoning behind them; this RFP applies them with the chains
swapped and does not restate them. Delivering this RFP against a materially
different set of patterns would fragment the two bridges for no benefit, so
RFP-021 is treated as a hard dependency rather than a reference.

#### Ethereum state attestation (RFP-022)

The inbound leg releases gas token only against an ERC-20 burn proven to have
been finalised on Ethereum, with no signer or federation trusted to attest to it
(Functionality #4). That verification is delivered as a shared primitive by
[RFP-022](./RFP-022-ethereum-state-attestation.md). This RFP consumes it and
does not rebuild it. The outbound leg, verifying a LEZ lock natively on
Ethereum, is specific to this bridge and remains in scope here.

#### Admin authority (RFP-001)

The Functionality requirements specify that an admin authority configures caps,
finality depth, and fee parameters. These admin-gated functions require the
standardised admin authority library from
[RFP-001](./RFP-001-admin-authority-lib.md).

#### Freeze authority (RFP-002)

The Bridge Security requirements specify a freeze authority able to pause
minting and/or release as a circuit breaker independent of the caps. This
requires the standardised freeze authority library from
[RFP-002](./RFP-002-freeze-authority-lib.md).

#### Token authorities (LP-0013)

The LEZ vault program escrows and releases native gas token under program
control. This requires the token authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md),
which is **closed** (delivered).

#### Logos Ethereum core module

The Ethereum side must use the Logos Ethereum core module, including its
verified proxy features.

#### Wallet SDK

The UI must use the wallet SDK from the Lambda Prize wallet-SDK work.

#### RISC0 zkVM

The bridge verifies proofs in-program on LEZ. Because LEZ itself runs on RISC0,
this is a LEZ-runtime dependency rather than a choice the proposal makes;
proposals must leverage mature RISC0 implementations (e.g.
[Zisk](https://github.com/risc0/zisk)) rather than building custom circuits.

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
  for third-party security audit, and particular care around mint authorisation
  paths
- Cryptographic primitives (Merkle proofs, consensus verification, state root
  inclusion proofs)
- LEZ program development, private-state programs, and on-chain proof
  verification
- Fee abstraction, relayer design, or account-abstraction style sponsored
  transactions, given the gas circularity this RFP has to solve
- Smart-contract security auditing (proof validation, replay attacks, reorg
  handling, cap bypass, unauthorised mint paths, privacy-leak analysis)
- Cross-chain system design and integration testing

## ⏱ Timeline Expectations

Estimated software delivery duration: **10–14 weeks**. The construction mirrors
[RFP-021](./RFP-021-wrapped-erc20.md) and consumes
[RFP-022](./RFP-022-ethereum-state-attestation.md) for the Ethereum-reading
half, so the patterns and much of the tooling are established rather than
invented here. The new work is the LEZ-side vault, the Ethereum-side mintable
ERC-20 with its authorisation path, and the gasless release mechanism, which is
the largest unknown. This excludes the third-party audit lead time required
before mainnet deployment (Supportability #8), which is typically procured and
scheduled separately.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- [RFP-004 — Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (states the atomic-deshield gas pattern that presupposes an already-funded
  user)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (same atomic-deshield assumption)
- [RFP-020 — RedStone Off-Chain Oracle Adaptor for LEZ](./RFP-020-redstone-oracle-adaptor.md)
  (reference for in-program proof verification cost measurement)
- [RFP-021 — Privacy-Preserving Wrapped ERC-20 and Ether Bridge for LEZ](./RFP-021-wrapped-erc20.md)
  (the primary flow this RFP inverts)
- [RFP-022 — Trustless Ethereum State Attestation for LEZ](./RFP-022-ethereum-state-attestation.md)
  (delivers the verification of finalised Ethereum state the release path
  consumes)
- [Appendix: Bridges and Wrapped Tokens](../appendix/bridges-and-wrapped-tokens.md)
  (bridge failure taxonomy, including the Meter Passport native-gas-token mint)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
- [LP-0013: Token program improvements: authorities](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md)
- [RISC0 — Zero-Knowledge VM](https://github.com/risc0/risc0)
- [Zisk — RISC0 Proof Generation](https://github.com/risc0/zisk)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
