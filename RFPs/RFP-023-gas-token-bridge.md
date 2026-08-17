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
the release. This RFP resolves it with an **off-chain paymaster service**, the
third component of a deployment alongside the vault and the Ethereum contract. A
user who has burned the ERC-20 drafts the release transaction locally and
contacts the paymaster over Logos Delivery or Tor; the paymaster checks the
attestation, submits the transaction, and pays the gas. It sponsors only burns
of its own deployment's ERC-20, never learns the requester's network address,
and can decline without blocking anyone.

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

The mechanism this RFP specifies is an **off-chain paymaster service**, deployed
alongside the vault and the Ethereum contract as the third component of a
deployment. A user who has burned the ERC-20 on Ethereum drafts the LEZ release
transaction locally, contacts the paymaster over an anonymising transport, and
the paymaster submits it and pays the gas. The complete deployer journey is
therefore: deploy the LEZ vault program, deploy the EVM mint and burn contract
with its dedicated ERC-20, and run the paymaster.

The eligibility rule is what keeps this from being an open faucet, and it is
expressed entirely in terms of the proof rather than the requester. The
paymaster sponsors a transaction if and only if that transaction carries a valid
attestation of a burn of **its own deployment's** ERC-20, at or above a
configured minimum amount, whose statement identifier has not already been
sponsored. Nothing about the requester enters the decision: no account, no
allowlist, no registration, no payment relationship. Someone who has not burned
the ERC-20 cannot obtain sponsorship, and someone who has burned it once cannot
obtain it twice, because the nullifier that already exists for double-spend
protection (Functionality #14) doubles as the anti-abuse budget. The cost of
abusing the paymaster is the cost of acquiring and burning real ERC-20, which is
the same barrier that protects the vault itself.

The paymaster is a liveness convenience, never a trust dependency. It cannot
redirect the release, because the destination is bound into the proof (Bridge
Security #2); it cannot forge or censor selectively without simply declining,
which any other paymaster instance can cover; and it holds nothing of the
user's. A deployment may run several, a user may try them in any order, and a
user who already holds gas can always submit the transaction themselves and skip
the paymaster entirely. Nothing in the protocol privileges a particular
paymaster or requires one to exist.

#### Transport, and why it needs care

The paymaster is contacted over **Logos Delivery**, the ecosystem-native
transport, with **Tor as a required alternative**. The alternative is not
redundancy for its own sake: Delivery's spam protection may itself require LEZ
gas for RLN, which the user by definition does not have, so a deployment that
offered only Delivery could reintroduce the very circularity this component
exists to break. Proposals must establish whether that dependency applies at
delivery time and document the finding either way; the Tor path must work
regardless, so the user always has a route that assumes nothing on LEZ.

Transport is where the privacy of this whole construction is most easily lost.
The paymaster necessarily learns the release transaction it is asked to submit,
which names the destination account. If it also learns the requester's IP, it
can link that account to a network identity, and the unlinkability the rest of
the design works to preserve is gone at the last step, in the one place where
the user has no choice but to talk to somebody. **The paymaster must therefore
never observe or record requester IP addresses.** Both transports are chosen for
this reason (Delivery and Tor each conceal the origin), and the requirement is
not satisfied by a promise not to log: the service must be built so that the
address is not available to it in the first place, and any deployment
configuration that would expose it, such as a plain HTTP fallback or a reverse
proxy passing an originating-address header, must be absent rather than merely
discouraged.

What the paymaster does learn is the destination account and the amount, at the
moment of submission. That is unavoidable for any party that submits on a user's
behalf, and it is why the operator's knowledge is an explicit documentation
deliverable (Supportability #9) rather than something to gloss over. It does not
link back to the Ethereum burn unless the transport leaks the requester, which
is what the IP requirement forecloses.

#### Griefing the paymaster

Eligibility is checked against the attestation, but *checking* it costs the
paymaster something, and on LEZ that cost may land before the check completes.
Fee reservation precedes execution and failed transactions are still charged, so
a paymaster that commits to paying before the proof is verified can be drained
by an attacker submitting well-formed garbage: each submission costs the
attacker nothing and costs the paymaster a reserved fee. This is not a
hypothetical concern about the design; it is the concrete blocker identified in
[`fryorcraken/lez-proof-vault`](https://github.com/fryorcraken/lez-proof-vault),
whose README works through why naive program-sponsored gas is unsafe under a fee
market and suggests deposit-and-reimburse as the alternative shape.

Proposals must therefore specify how the paymaster bounds this cost, and the
choice is theirs: verifying the attestation off-chain before submitting anything
on-chain, so the paymaster spends nothing on an invalid request; a cheap
reservation-time check that rejects the bulk of garbage before the expensive
path; a deposit-and-reimburse construction where the submitter fronts the fee
and is repaid out of the release; or per-transport rate limiting that does not
require identifying the requester. Whatever is chosen must not reintroduce a
requester identity, since that would defeat the transport privacy above, and
must be measured rather than asserted (Performance #7).

That repository is the closest prior art for the LEZ-side vault and its
proof-submission shape, and is useful reading for the ownership-versus-
authorization framing of a program-owned vault and for the claimant, submitter
and payer role split it sets out. It is prior art and not a specification: it
implements no paymaster, no transport, and no burn or mint flow, its proof is a
SHA-256 preimage standing in for a real proof system, and it documents its own
gaps (no recipient binding, no authority binding on initialisation). Proposals
should not treat it as a pattern to conform to.

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
    and no specific paymaster or other off-chain party may be a required
    counterparty (see Design Rationale, "The gas circularity"). Test the release
    path end to end with a recipient account holding a zero balance.
05. **Paymaster service.** Provide an off-chain paymaster that accepts a drafted
    LEZ release transaction, submits it, and pays the gas, so that a user with a
    zero balance can complete a release. It must be implemented as a **Logos
    module accompanied by a Logos Core headless CLI/daemon**, runnable
    standalone by the deployer of a vault and contract pair.
06. The paymaster sponsors a request if and only if the transaction carries a
    valid attestation of a burn of its own deployment's ERC-20, at or above a
    configured minimum amount, whose statement identifier it has not already
    sponsored. Eligibility must depend on no property of the requester: no
    account, allowlist, registration, or payment relationship. Test that a
    request carrying no burn, a burn of a different token or deployment, a burn
    below the minimum, or an already-sponsored burn is refused, and that a
    request carrying a valid unsponsored burn is accepted regardless of who
    sends it.
07. The paymaster is reachable over **Logos Delivery** and over **Tor**, and a
    user must be able to complete a release using either transport alone.
    Proposals must establish whether Logos Delivery's spam protection requires
    LEZ gas for RLN at delivery time and document the finding; the Tor path must
    function regardless, so that a user holding nothing on LEZ always has a
    working route.
08. **The paymaster must not observe or record requester IP addresses.** This
    must hold by construction rather than by logging policy: the service must
    have no configuration, deployment shape, or transport fallback through which
    an originating address becomes available to it. Provide a test asserting
    that no requester address is present in any log, metric, persisted record,
    or in-memory request context, and document the deployment constraints that
    preserve this (see Privacy Preservation #7).
09. Sponsorship is refused without revealing why in a way that identifies the
    burn: refusal responses for an ineligible request, an already-sponsored
    burn, and a paymaster out of funds must be indistinguishable in that
    respect.
10. Nothing in the protocol may privilege a particular paymaster. A deployment
    must support running several, a user must be able to select or switch
    between them per operation, and a user who holds gas must be able to submit
    the release themselves with no paymaster involved. Test the release path
    with no paymaster running.
11. The paymaster's exposure to invalid or repeated requests must be bounded, so
    that an attacker submitting well-formed but ineligible requests cannot drain
    it (see Design Rationale, "Griefing the paymaster"). Proposals must state
    the chosen mechanism, which must not require identifying the requester.
    Provide a test that floods the paymaster with ineligible requests and
    asserts a bounded cost per request and continued service to a valid one.
12. A burn on Ethereum must not publish, store, or otherwise reveal its LEZ
    destination.
13. Releasing must support both a private LEZ account and a public LEZ account
    as the destination, at the burner's choice.
14. Uniqueness is enforced in both directions: no lock can be minted against
    twice and no burn released against twice, deterministically and under
    adversarial retry. On the LEZ side this is keyed on the statement identifier
    the RFP-022 attestation carries.
15. The amounts visible on Ethereum must not identify which lock or release they
    correspond to. Proposals must state the mechanism chosen (fixed
    denominations are the expected baseline) and its effect on anonymity-set
    size.
16. A user must be able to recover every one of their own unclaimed locks and
    unreleased burns from credentials they already hold, with no dependence on
    any server-side index and no separately-backed-up secret generated during
    the flow, on the same terms as RFP-021, "Loss of access."
17. An admin authority (per [RFP-001](./RFP-001-admin-authority-lib.md),
    integrated via the [SPEL framework](https://github.com/logos-co/spel) where
    applicable to the LEZ side) can configure the caps, the finality depth, and
    the fee parameters per deployment.
18. Global caps, configurable by the admin authority, bound the maximum value
    that can be minted or released within a rolling window, as a rate limiter
    independent of the freeze authority. Cap enforcement must not require
    identifying individual users.
19. The finality depth required before a lock may be minted against, and before
    a burn may be released against, is configurable by the admin authority per
    deployment. A change to the configured depth must not invalidate a claim
    that was already valid under the previous depth.
20. A freeze authority (per [RFP-002](./RFP-002-freeze-authority-lib.md)) can
    pause minting and/or release, on the Ethereum contract and the LEZ vault
    program independently.
21. Each LEZ vault program deployment refers to a specific Ethereum contract
    deployment on a specific chain (contract address plus chain ID), and
    reciprocally each Ethereum contract deployment refers to a specific zone
    instance (LEZ blockchain ID, zone ID, and program ID), with each pairing
    checked as part of proof verification so a lock or burn valid for one
    pairing is never accepted as valid for another. The same program and
    contract design must be deployable, unmodified, against any EVM chain,
    mainnet or testnet.
22. The design must let multiple entities each operate under their own
    independent configuration (caps, fees, admin authority, finality depth), on
    the same or different pairs of blockchain programs, with strict separation
    between them: one entity's configuration must have no privileged access over
    another's configuration or funds. Document how a client identifies and
    switches between configurations.
23. A protocol fee may be charged on minting and on release, at a rate
    configurable by the admin authority per deployment, including zero. The fee
    value must not distinguish a user's transaction from others, consistent with
    Functionality #15. Where the release-path mechanism reimburses a paymaster
    out of the released amount, document how that reimbursement interacts with
    the fixed-denomination requirement.

#### Usability

01. Build core functionalities for both users and admin in a Logos core module,
    enabling the delivery of different Logos ui modules: locking gas token,
    claiming the Ethereum mint, burning the ERC-20, claiming the LEZ release,
    recovering a position from user credentials, and reading and administering
    the configuration.
02. Provide a Logos mini-app, aka Logos ui module, covering both flows end to
    end, position recovery, and a view showing permitted amounts, caps and
    current utilisation. Also provide a UI for the admin functionality; whether
    this is combined into one UI or delivered as two separate ones is left to
    the applicant's choice.
03. The onboarding flow must be usable by someone who holds nothing on the zone.
    The mini-app must not require a funded LEZ account to complete a release,
    and must not present a step that silently assumes one.
04. Any long-running off-chain component the design requires, including the
    paymaster, must be provided as a **Logos module accompanied by a Logos Core
    headless CLI/daemon**, runnable standalone, supporting configurable RPC
    endpoints for both chains, configurable finality depth, structured logging,
    and a clean shutdown path. Document the operator journey end-to-end:
    install, configure, run, monitor.
05. The client must let the user select which paymaster to contact and over
    which transport, retry against another on refusal or timeout, and fall back
    to self-submission when the user holds gas. A paymaster refusing or being
    unreachable must produce a clear, actionable state rather than a stalled
    flow.
06. Provide an IDL for the LEZ vault program using the
    [SPEL framework](https://github.com/logos-co/spel).
07. The mitigations to the three correlation points in Design Rationale,
    "Privacy mirrors RFP-021 exactly" (amount, timing, fee payer) must be
    enabled by default. The mini-app and CLI must show a clear indicator of what
    data would be leaked by the user's current choices, and default to the
    recommended parameters rather than requiring the user to select them.
08. The mini-app and CLI must default to inviting the user to release into, and
    lock from, a private account: the private path is the pre-selected option,
    and choosing the public path requires an explicit action, consistent with
    Privacy Preservation #8.
09. Documentation and UI must clearly explain what is public and what is private
    at each step on both chains, and must set the expectation described in
    Design Rationale, "Supply is demand-driven", that the ERC-20 is available
    only to the extent someone has moved gas token outward.
10. Return clear, actionable error messages for all failure modes: invalid
    amount, cap exceeded, verification failure, insufficient finality, already
    claimed, and program frozen. Error messages must not reveal which lock or
    burn a failed attempt referred to.

#### Reliability

01. Minting is atomic: a failed or rejected mint claim leaves the lock claimable
    on retry and consumes nothing.
02. Release is atomic: a failed burn does not destroy the ERC-20 without
    preserving the holder's entitlement to release, and a failed release leaves
    that entitlement intact.
03. No lock can be minted against twice and no burn released against twice,
    deterministically and under adversarial retry.
04. A valid claim remains valid indefinitely; later chain activity must never
    invalidate a user's outstanding entitlement.
05. Position recovery is complete: a client restored from user credentials alone
    must rediscover every claimable lock and unreleased burn, verified by a test
    that wipes all local state.
06. Temporary RPC or connectivity failure on either chain leaves any off-chain
    component in a recoverable state, able to resume without duplicating work
    already done.
07. An interrupted user-side operation does not consume, corrupt, or expose the
    user's entitlement. A paymaster that accepts a request and then fails,
    crashes, or never submits must leave the user's entitlement intact and
    re-submittable, to the same or another paymaster, with no state stranded on
    the failed one.
08. No paymaster is required for correctness. With every paymaster offline, a
    user holding gas must still be able to complete a release themselves, and a
    user without gas must be left in a recoverable state rather than losing the
    entitlement. Test with no paymaster reachable over either transport.
09. Proposals must integrate mature, audited proof-system implementations rather
    than reimplementing zero-knowledge primitives from scratch.
10. CI must be green on the default branch.

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
   on-chain verification, and the paymaster round trip over each supported
   transport, since Delivery and Tor differ materially in latency.
5. Document the compute resources (CPU, RAM, time) required to run any off-chain
   component the design requires, including the paymaster.
6. Document the growth rate and on-chain storage cost of all bridge state that
   accumulates with usage, with projections at 1M and 10M operations.
7. **Measure the paymaster's cost per request**, separately for an eligible
   request and for each class of rejected request, and state the resulting bound
   on what an attacker can force the paymaster to spend per unit of their own
   cost (see Design Rationale, "Griefing the paymaster"). Document the operating
   budget a paymaster needs at a stated request volume, and the behaviour when
   its funds are exhausted.

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
    independently configured deployment (Functionality #22) as a complete set:
    deploying the LEZ vault program, deploying the EVM mint and burn contract
    with its dedicated ERC-20, and running the paymaster. The paymaster section
    must cover funding it, configuring the minimum sponsored amount, exposing it
    over Logos Delivery and Tor, and the deployment constraints that keep
    requester addresses out of reach (Functionality #8).
08. The Ethereum contract undergoes an independent third-party smart-contract
    security audit before mainnet deployment; the audit report must be
    published. The audit scope must explicitly include the mint authorisation
    path, given the Meter Passport precedent in Design Rationale.
09. Provide a **privacy properties document** on the same terms as RFP-021,
    Supportability #9, covering: a formal statement of Privacy Preservation #1
    and #2 and the anonymity set each is measured against; exactly what is
    visible on-chain at every step on both chains; what an adversary observing
    all public state can and cannot infer; what every off-chain participant can
    observe, with a specific section on the paymaster stating exactly what it
    learns (the destination account and amount at submission time), what it
    cannot learn (the requester's network address, and the link back to the
    Ethereum burn), and what a malicious or compromised paymaster could and
    could not do; residual leakage from timing, amount selection, fee payment,
    network metadata and usage patterns, including what each supported transport
    exposes; and the conditions under which the guarantees degrade.
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
   This applies with particular force to the release path, where a paymaster is
   submitting on behalf of a user who cannot submit for themselves.
3. Caps (Functionality #18) bound the maximum value at risk in any rolling
   window; proposals must document recommended defaults and the reasoning.
4. The freeze authority (Functionality #20) must be exercisable independently on
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
7. **The paymaster must not become a correlation signal.** Whoever pays for a
   release must not thereby link the Ethereum burn to the LEZ account receiving
   the gas. Concretely: the paymaster must not observe or record requester IP
   addresses (Functionality #8), must not require or accept any requester
   identifier, and its own on-chain footprint as fee payer must not distinguish
   one sponsored release from another. Document precisely what it learns, ensure
   the user can switch between paymasters and transports per operation, and
   provide a test asserting that neither the paymaster's identity nor its
   payment narrows the anonymity set. This is the requirement most at risk from
   the gas circularity, since it is the one point where the user must talk to
   somebody, and must be treated as a primary design constraint rather than a
   late mitigation.
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
   logic (fixed-denomination handling, position recovery, submission paths, the
   privacy test harness), factor it so both can consume one implementation
   rather than maintaining two divergent copies. Document what is shared and
   what is necessarily distinct.

4. **Optional viewing keys** allowing a user to *voluntarily* disclose their own
   bridge activity to a chosen third party, without weakening privacy for anyone
   else and without any protocol-level disclosure capability.

5. **Additional EVM chains**, each served by its own deployment per
   Functionality #21 (one program per chain ID, not one program juggling several
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
- **Building new anonymising network infrastructure.** The paymaster must be
  reachable over Logos Delivery and Tor and must not observe requester addresses
  (Functionality #7, #8), but this RFP integrates existing transports rather
  than designing a mixnet or hardening the transports themselves. Residual
  network-level leakage outside the paymaster path, such as which Ethereum RPC
  or LEZ sequencer a user's client contacts, stays an implementation concern and
  must be disclosed under Supportability #9 rather than solved here.
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

#### Logos Delivery

The paymaster is reachable over Logos Delivery as its ecosystem-native transport
(Functionality #7). Tor is a required alternative and the release path must work
over Tor alone, so Delivery being unavailable never blocks a user. Proposals
must establish whether Delivery's RLN spam protection requires LEZ gas at
delivery time, since a user in the motivating case has none, and document the
finding either way. Following [RFP-003](./RFP-003-atomic-swaps.md), the
application must handle Delivery being temporarily unreachable gracefully and
must not depend on it to complete an operation already in progress.

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
- Fee abstraction, paymaster or relayer design, or account-abstraction style
  sponsored transactions, given the gas circularity this RFP has to solve
- Anonymising transports (Tor hidden services, mixnets, Logos Delivery) and
  building services that are structurally unable to observe client network
  addresses
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
- [RFP-003 — Atomic Swaps](./RFP-003-atomic-swaps.md) (precedent for using Logos
  Delivery for coordination without depending on it for completion)
- [Appendix: Bridges and Wrapped Tokens](../appendix/bridges-and-wrapped-tokens.md)
  (bridge failure taxonomy, including the Meter Passport native-gas-token mint)
- [`fryorcraken/lez-proof-vault`](https://github.com/fryorcraken/lez-proof-vault)
  (prior art for a program-owned LEZ vault releasing on proof; documents the
  reserve-before-verify griefing problem for sponsored gas)
- [LP-0012: Event/Log mechanism for LEZ](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
- [LP-0013: Token program improvements: authorities](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0013.md)
- [RISC0 — Zero-Knowledge VM](https://github.com/risc0/risc0)
- [Zisk — RISC0 Proof Generation](https://github.com/risc0/zisk)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
