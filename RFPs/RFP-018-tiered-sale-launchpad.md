---
id: RFP-018
title: "Privacy-Preserving Token Launchpad: Tiered Sale"
tier: L
funding: $XXXXX
status: open
dependencies: See Platform Dependencies section
category: Applications & Integrations
---


# RFP-018: Privacy-Preserving Token Launchpad: Tiered Sale

Note that ecosystem research is available in the [appendix](appendix/token-launchpad-ecosystem).

## 🧭 Overview

Build a tiered sale token launchpad on LEZ, a fixed-price token
distribution mechanism with multiple sequential rounds (e.g., Seed,
Private, Public), each with independently configurable prices,
allocation sizes, and access restrictions. Participants can buy
tokens directly from a public account or, for privacy, via a
deshield→buy→re-shield pattern (see
[RFP-008](./RFP-008-lending-borrowing-protocol.md), which defines this
interaction model for LEZ applications): when a private account is
used, each purchase is routed through a fresh, single-use public
account, hiding the link between the buyer's private identity and their
participation in the sale. An optional private allowlist gate per tier
enables projects to restrict participation without exposing the
eligibility set on-chain.

The tiered sale launchpad complements the Logos ecosystem's existing
token launch toolkit: where bonding curves
([RFP-015](./RFP-015-bonding-curve-launchpad.md)) reward early
believers with supply-driven pricing and LBPs
([RFP-016](./RFP-016-lbp-launchpad.md)) run time-driven price
discovery, tiered sales serve projects that already have a known
valuation and want controlled distribution across investor tiers at
predetermined prices.

## 🔥 Why This Matters

Tiered sales are an established distribution mechanism for projects
with an existing investor base and a known target valuation. Fjord
Foundry supports tiered sales (Seed, Private, and Public rounds) with
per-tier configuration of token amounts, prices, and allocation limits
[33]; across all sale types (primarily LBPs), the platform has hosted
717 sales with over $1.1B in cumulative funds raised and 106,762
participants since 2021 [5][6]. DAO Maker's Strong Holder Offering
(SHO) model uses phased rounds (priority tranches for high-tier
holders, then public rounds) and has raised $90M+ [8][12]. Polkastarter runs
fixed-price IDOs with hard caps across 105+ projects and $46M+ in
cumulative funds raised [10]. Metaplex Genesis on Solana supports
multiple inflow buckets per launch, enabling private/seed rounds
(allowlisted wallets, fixed price) followed by public rounds (open,
fair launch pool), with an estimated $55-70M in total funds raised
during its alpha period [16].

On transparent chains, participation in a tiered sale is immediately
linkable to a wallet address and, by extension, to the buyer's full
transaction history. Early investors in seed and private rounds are
especially exposed: their wallet addresses, allocation sizes, and
vesting unlock dates are all public, making them targets for social
engineering and front-running. Platforms like Fjord Foundry, DAO Maker,
and Polkastarter all require participants to expose their wallet
identities on-chain; those using KYC leak that data to a centralised
service [11][12][10].

On LEZ, the optional deshield→buy→re-shield pattern means a buyer's
private account is never linked to their purchase on-chain. Observers
see a buy transaction from an ephemeral public account with no prior
history. The buyer's identity, their tier allocation, and the
destination of their purchased tokens are all private. This is a
structurally different, and stronger, privacy guarantee than anything
available on existing tiered sale platforms.

## 🏗 Design Rationale

### Tiered sales and their place in the ecosystem

A tiered sale distributes tokens at predetermined fixed prices across
multiple sequential rounds, each with its own access rules, allocation
limits, and token quantity. The mechanism is well suited to projects
that have already established a valuation through prior fundraising,
community building, or market analysis and want to distribute tokens
in a controlled, phased manner to different investor classes.

Tiered sales differ from the other two launchpad mechanisms in the
Logos ecosystem in a fundamental way: they do not perform price
discovery. A bonding curve
([RFP-015](./RFP-015-bonding-curve-launchpad.md)) discovers price
through supply progression, with each purchase moving the price higher.
An LBP ([RFP-016](./RFP-016-lbp-launchpad.md)) discovers price
through time-driven weight shifting, letting the market converge on
fair value over days. A tiered sale, by contrast, fixes the price per
tier before the sale opens. This makes tiered sales the right choice
when the project already knows its target price and wants deterministic
raise outcomes, and the wrong choice when price discovery is needed.

The three mechanisms together give the Logos ecosystem a complete token
launch toolkit: bonding curves for early-stage projects that want to
reward early believers, LBPs for projects seeking broad community price
discovery, and tiered sales for projects with established valuations
distributing across investor tiers.

### Fixed price per tier

Each tier has a predetermined token price set by the sale creator at
creation time. The price does not change during a tier's active window.
This is the standard model across the ecosystem: Fjord Foundry's tiered
sales support per-tier price configuration [33], DAO Maker's SHO
allocates at a fixed price per round [12], and Polkastarter IDOs use
a single fixed price per sale [10].

Fixed pricing eliminates price uncertainty for both the project and
participants. Projects know exactly how much collateral they will raise
at each tier (assuming full subscription), and participants know their
exact entry price before committing. The tradeoff is that mispricing
is binary: if the price is set too high, the tier will not fill; if
set too low, the project leaves value on the table. Projects that want
the market to find the right price should use
[RFP-016](./RFP-016-lbp-launchpad.md) instead.

### Minimum cap with auto-refund

A minimum raise target can be configured for the entire sale. If the
total collateral raised across all tiers does not reach this minimum
by the time the final tier closes, all contributions are automatically
refunded to participants. Fjord Foundry's tiered sales support this
feature: if the minimum cap is not met, contributions are automatically
refunded [33]. Metaplex Genesis's Launch Pool also refunds depositors
if the funding goal is not met [16], and DAOs.fun provides full refunds
when the fundraise hard cap is not reached [21].

The auto-refund mechanism protects both participants and projects. It
prevents a project from launching with insufficient capital, and it
assures participants that their funds will be returned if the raise
fails to reach a viable threshold.

### Per-wallet allocation limits

Unlike the LBP and bonding curve launchpads (which cannot enforce
per-wallet limits for the private account path because each buy uses a
fresh ephemeral address), the tiered sale can enforce per-wallet
allocation limits for public account interactions. Each tier can
specify a minimum and maximum allocation per participant.

For the private account path, per-wallet limits cannot be enforced
on-chain for the same reason as in RFP-015 and RFP-016: each buy from
a private account uses a fresh ephemeral public address, so the program
cannot link multiple buys to the same underlying participant. The
allowlist gate with limited entries per eligibility proof is the
recommended mitigation for the private account path: each allowlist
entry can be restricted to a single use via a nullifier, bounding total
allocation per eligible participant regardless of how many ephemeral
accounts they use.

### Public sale state

All tiered sale state is public on-chain (tier configuration, prices,
allocation limits, collateral raised per tier, open/closed status per
tier). As with [RFP-015](./RFP-015-bonding-curve-launchpad.md) and
[RFP-016](./RFP-016-lbp-launchpad.md), this is a deliberate
architectural choice. Public state enables permissionless price
verification, composability with other LEZ programs, and verifiable
sale analytics without cryptographic complexity in the sale program
itself. Participant privacy is enforced at the UX layer via the
optional deshield→buy→re-shield pattern, which does not require private
sale state.

### Optional allowlist

The allowlist gate is opt-in and configurable per tier: a seed tier
might require allowlist membership while the public tier is open to
all. When gating is needed (geographic restrictions, community-only
rounds, pre-selected investor lists), the creator can commit an
eligibility set at creation time and restrict buys to participants who
can prove inclusion. The implementation approach is left to the
proposing team.

On LEZ, the relevant anonymity set for any private account action is
all private accounts in the zone, not the allowlist size. The
execution environment is shielded by construction: an observer cannot
determine which private account interacted with the sale, regardless
of whether an allowlist is active. This means an allowlist does not
degrade privacy in the way it would on a transparent chain, where
the anonymity set shrinks to the allowlist size (see the
[Allowlist Privacy in Shielded Execution Environments](../appendix/token-launchpad-ecosystem.md#allowlist-privacy-in-shielded-execution-environments)
section of the token launchpad ecosystem appendix).

The recommended approach is ZK set membership proofs (a Merkle tree
commitment with a ZK inclusion proof) rather than a public address
list. This avoids publishing the allowlist on-chain and preserves the
zone-wide anonymity set regardless of allowlist size. When both the
allowlist gate and the private account path are enabled, the proposal
must document the resulting privacy properties and any reduction in
anonymity set relative to a non-gated sale.

### Sale pause capability

The pause function is an emergency stop for security incidents,
analogous to the freeze authority in
[RFP-002](./RFP-002-freeze-authority-poc.md). Pausing halts all
buy operations across all tiers. The pause does not affect tier
scheduling: if a tier's time window expires during a pause, it remains
closed when the sale resumes. This prevents a creator from gaming the
mechanism by pausing at a moment that is artificially favourable and
extending a tier's effective duration.

### Fee structure

This RFP does not mandate a specific fee rate. Proposals must
specify: (1) who pays the fee (issuer, buyer, or both), (2) when
fees are collected (per-transaction, at sale close, or both),
(3) the fee rate or rate range, and (4) where fees are routed
(protocol treasury, sale creator, burn, or other destination). See
the [Fee Structures](../appendix/token-launchpad-ecosystem.md#fee-structures)
section of the token launchpad ecosystem appendix for a
cross-platform comparison of fee models across seven surveyed
protocols.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a tiered sale program on LEZ. A tiered sale consists of
   one or more tiers (rounds), each with a fixed token price, a token
   allocation, and an independent time window. Tiers execute
   sequentially: a tier becomes active at its start timestamp and
   closes at its end timestamp. No two tiers may have overlapping
   active windows.
2. A sale creator can configure a sale with the following parameters:
   1. Collateral token (e.g., stablecoin or native token), shared
      across all tiers.
   2. Project token, shared across all tiers. All project tokens for
      all tiers must be deposited by the creator at sale creation.
   3. Per-tier configuration (one or more tiers), each with:
      - Tier name or identifier (e.g., "Seed", "Private", "Public").
      - Token price (fixed, denominated in the collateral token).
      - Token allocation (number of project tokens available in this
        tier).
      - Start and end timestamps.
      - Optional: per-wallet minimum and maximum allocation limits
        (enforced for public account interactions; see Design
        Rationale for private account path limitations).
      - Optional: private allowlist gate (see item 8 below),
        configurable independently per tier.
   4. Optional: minimum raise target (minimum total collateral across
      all tiers). If set, and the minimum is not met by the time the
      final tier closes, all contributions are automatically refunded
      (see item 6 below).
3. Participants buy project tokens from an active tier using either a
   public account directly, or via the deshield→buy→re-shield
   pattern for private account interaction (see
   [RFP-008](./RFP-008-lending-borrowing-protocol.md)). Both paths
   must be supported by the program and SDK.
4. A buy transaction specifies the collateral amount to spend. The
   program computes the token output using the tier's fixed price.
   If the requested amount would exceed the tier's remaining token
   allocation, the transaction must revert (no partial fills).
5. After the final tier closes (and the minimum raise target is met,
   if configured), the creator can withdraw:
   - The collateral raised across all tiers, net of fees as defined
     by the fee model specified in the proposal (see the Fee
     structure subsection in Design Rationale).
   - Any unsold project tokens remaining across all tiers.
6. Minimum raise with auto-refund: if a minimum raise target is
   configured and the total collateral raised across all tiers does
   not meet this target by the time the final tier closes, the sale
   enters a refund state. In the refund state: (a) the creator
   cannot withdraw collateral, (b) each participant can reclaim
   their full collateral contribution, and (c) the creator can
   reclaim all deposited project tokens once all participant refunds
   have been processed or after a configurable claim window expires.
7. The sale creator can pause buying at any time during the sale
   period (emergency stop). Pausing halts all buy operations across
   all tiers. Tier time windows continue to advance during a pause;
   if a tier's end timestamp passes while the sale is paused, that
   tier is closed when the sale resumes.
8. The sale creator can enable an optional allowlist gate per tier.
   When enabled for a tier, only participants who can prove inclusion
   in the committed eligibility set may buy from that tier. Different
   tiers can have different allowlists (e.g., a seed tier restricted
   to investors, a public tier open to all). The proposing team must
   specify and justify their allowlist mechanism in their application.
   When the allowlist gate is used in conjunction with the private
   account path, the proposal must document the resulting privacy
   properties, including any change in the effective anonymity set
   relative to a non-gated sale.
9. Allocation limit enforcement: for public account interactions,
   the program must track cumulative spend per address per tier and
   enforce the per-wallet minimum and maximum limits. A buy that
   would cause the participant's cumulative allocation to exceed the
   tier's maximum must revert. A buy below the tier's minimum must
   also revert, unless the remaining tier allocation is less than
   the minimum (in which case the minimum is waived to allow the
   tier to fill completely).
10. Use Associated Token Accounts (ATAs) for all token interactions,
    consistent with
    [LP-0014](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
    and [RFP-008](./RFP-008-lending-borrowing-protocol.md).

#### Usability

1. Provide an SDK for building Logos modules that interact with the
   tiered sale program. The SDK must expose the full lifecycle for
   both participants (discover active sales, query tier status, buy,
   query allocation, claim refund) and creators (create sale,
   pause/resume, close, withdraw, configure tiers). The SDK must
   support both direct public account interaction and the
   deshield→buy→re-shield pattern for private account interaction.
   When the private account path is used, the SDK must handle the
   atomic deshield (both collateral and gas) as a single
   indivisible user action.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo. The mini-app must cover:
   - **Participant view**: browse active sales with per-tier details
     (price, allocation remaining, time window, allowlist status),
     execute a buy, view purchase history, claim refund (if
     applicable).
   - **Creator view**: create a new sale (all parameters including
     per-tier configuration and allowlist setup), monitor an active
     sale (live tier progress, collateral raised per tier and total),
     pause/resume, close sale, and withdraw proceeds.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations for both participants (buy,
   query price, query tier status, check sale status, claim refund)
   and creators (create sale, pause/resume, close, withdraw).
4. The mini-app must display a pre-buy confirmation summary before
   each purchase: tier name, token price, exact tokens to be
   received, collateral to spend, remaining tier allocation, and fee.
5. When using the private account path, the mini-app must display a
   privacy disclosure before each buy, identifying what will be
   visible on-chain (buy transaction, collateral amount, tokens
   received, tier, sale address, ephemeral account) and what will
   not be traceable (the buyer's private account and the subsequent
   destination of purchased tokens).
6. When using the private account path, the mini-app must enforce
   the atomic deshield, preventing a buyer from funding the
   ephemeral account from any external source, which would create
   an on-chain link to an existing identity.
7. When using the private account path, the mini-app must confirm
   before each buy that the buyer's shielded balance covers both
   the collateral amount and the gas fee within the single deshield
   action. A clear, actionable error must be shown if the balance
   is insufficient, preventing a partial deshield that could leave
   funds stranded.
8. Provide a sale analytics view showing, for each active or
   completed sale: total collateral raised (per tier and overall),
   tier fill rates (percentage of allocation sold per tier), number
   of buy transactions per tier, and time-to-fill for completed
   tiers. Analytics must not expose individual participant identities
   or link buy transactions to specific accounts.
9. Provide an IDL for the tiered sale program using the
   [SPEL framework](https://github.com/logos-co/spel).
10. Failed or rejected buys must return clear, actionable error
    messages (e.g., insufficient balance, tier not yet active, tier
    ended, tier fully allocated, allowlist gate rejected, allocation
    limit exceeded, sale paused, minimum not met for refund
    eligibility).

#### Reliability

1. Sale state (collateral raised per tier, tokens remaining per tier,
   cumulative allocations per participant) must remain consistent
   under concurrent buy submissions. No double-spend or incorrect
   sale accounting.
2. A failed buy must revert atomically: the buyer's collateral is
   not consumed and the sale state is unchanged.
3. Refund state transition must be atomic: when the final tier closes
   and the minimum raise target is not met, the sale must transition
   to the refund state in the same transaction that processes the
   final tier's close. No additional instruction should be required
   to trigger the refund state.
4. Refund claims must be idempotent: a participant who has already
   claimed a refund must not be able to claim again. The program must
   track refund status per participant.

#### Performance

1. A single buy transaction completes within one LEZ transaction.
2. A refund claim completes within one LEZ transaction.
3. Document the compute unit (CU) cost of each operation: create
   sale, buy, pause/resume, close sale, withdraw, claim refund.
   Note the LEZ testnet version against which measurements were
   taken.

#### Supportability

Proposals must include separate milestones for testnet 0.2, testnet 0.3,
and mainnet deployment.

1. The tiered sale program is deployed and tested on LEZ testnet 0.2.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI. CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test. Test
   coverage must include: happy-path buy across multiple tiers,
   tier allocation exhaustion, per-wallet limit enforcement (accept
   and reject), allowlist gate accept and reject, minimum raise met
   (creator withdrawal), minimum raise not met (refund flow),
   sale pause during active tier, tier time window expiry.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for both creators and
   participants via CLI and mini-app.
5. Provide a privacy and anonymisation properties document covering:
   what on-chain state and transaction data is visible to observers;
   what data is protected when the private account path is used;
   trust assumptions, specifying which guarantees are enforced by
   the on-chain program and which depend on correct client
   behaviour; and what happens if a user bypasses the expected
   interaction path.
6. The program is updated and verified on LEZ testnet 0.3.
7. The program is deployed to LEZ mainnet.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→buy→re-shield pattern for private
   account interaction. When a user chooses the private account
   path, the SDK must enforce the complete deshield→buy→re-shield
   pattern; the re-shield step must not be skippable.
2. When using the private account path, the mini-app must display
   the pre-confirmation privacy summary described in the Usability
   section before each buy operation, identifying what is visible
   on-chain (buy amount, tier, sale address, ephemeral intermediary
   account) and what remains private (the originating private
   account, the destination of re-shielded tokens, and any link
   between separate buys by the same user).
3. When using the private account path, the SDK must validate that
   the re-shield target for purchased tokens is a private (shielded)
   account before submitting the transaction, and reject with an
   explicit error if it is not.
4. The ephemeral public account created during the deshield step
   must never be reused across operations. Each buy from a private
   account must use a freshly generated account with no prior
   on-chain history.

### Privacy Architecture

All tiered sale state is public on-chain (tier configuration, prices,
allocation limits, collateral raised per tier, tokens remaining per
tier, open/closed status per tier, minimum raise target, refund state).
This is a deliberate architectural choice: public state enables
permissionless price verification (participants can confirm the
exact price and allocation before buying), composability with other
LEZ programs, and verifiable sale analytics without cryptographic
complexity in the sale program itself.

User privacy is optionally enforced at the UX layer. The mini-app
and SDK support both direct public account interaction and private
account interaction via the deshield→buy→re-shield pattern. When
users opt to interact from a private account, the SDK must enforce
the complete pattern as described below.

#### Interaction flow

For every buy from a private account:

1. The buyer initiates from their private account. The SDK
   deshields to a **fresh, single-use** public account (account A)
   with no prior on-chain history. The deshield atomically
   transfers both the collateral token and enough native token for
   gas in a single indivisible action.
2. Account A executes the buy against the active tier.
3. Account A re-shields the purchased project tokens to the buyer's
   private account. Account A is never reused.

> **Gas:** Both collateral and gas must come exclusively from the
> deshield in step 1. Funding account A from any external source
> (such as a CEX withdrawal or a known wallet) creates an
> on-chain link to an existing identity and breaks the privacy
> guarantee. The SDK must make this impossible; the atomic deshield
> is a single, indivisible user action.

#### What is public (observable on-chain)

- All sale state: token pair, tier configuration (names, prices,
  allocations, time windows), collateral raised per tier, tokens
  remaining per tier, minimum raise target, sale status.
- All buy transactions: collateral spent, tokens received, tier,
  and timestamp. When using the private account path, the buyer's
  address is an ephemeral intermediary account with no prior
  on-chain history.
- Allowlist gate configuration per tier and whether the gate is
  enabled, but not the list of eligible addresses.
- Sale close, creator withdrawal, and refund transactions.

#### What is private (when using the private account path)

- Which private account originated the collateral for a buy.
- Where purchased tokens go after re-shielding.
- Any link between multiple buys by the same buyer (no on-chain
  linkability across ephemeral accounts).
- Whether a specific private account participated in the sale
  at all.
- Which tier a specific private account participated in (buys
  across tiers from the same private account use different
  ephemeral accounts).

#### Trust assumptions

The privacy guarantees above depend on the buyer using an SDK or
client that correctly implements the full deshield→buy→re-shield
pattern. The tiered sale program itself cannot enforce the re-shield
step: a buyer who calls the program directly or uses a non-conforming
client could skip the re-shield, leaving purchased tokens in the
ephemeral public account and breaking their own anonymity. The
program enforces correctness of the buy (price computation, tier
validation, allocation limits); the privacy pattern is enforced by
the SDK, not by the on-chain program. Buyers who bypass the SDK
accept full responsibility for any resulting privacy loss.

### Known Limitations

**Per-wallet allocation limits are not enforceable for the private
account path.** Enforcing a cap on cumulative spend per participant
requires tracking spend per address. For public account interactions
this is straightforward, but is fundamentally incompatible with the
private account path: each buy from a private account uses a fresh
ephemeral public address with no prior history, so the program
cannot link multiple buys to the same underlying participant. The
allowlist gate with nullifier-based single-use entries is the
recommended mitigation: each allowlist entry permits one buy per
tier, bounding total allocation per eligible participant regardless
of how many ephemeral accounts they use. Projects that need strict
concentration limits for all participants should use the allowlist
gate on all tiers.

### Soft Requirements

- **Buyer token vesting**: at sale close, purchased tokens can be
  routed directly into a vesting schedule (see
  [RFP-017](./RFP-017-token-vesting.md)) rather than transferred
  to the buyer immediately, enabling a sale-plus-vesting flow in a
  single operation.
- **Partial fills**: instead of reverting when the remaining tier
  allocation is less than the requested buy amount, the program
  fills the order partially (selling only the remaining tokens) and
  returns the excess collateral. This improves UX for the last buyer
  in a tier.
- **Creator collateral vesting**: the sale creator can configure an
  optional vesting schedule for the raised collateral at sale
  creation time. When configured, the close or withdraw operation
  deposits collateral into the vesting program (see
  [RFP-017](./RFP-017-token-vesting.md)) instead of transferring
  it directly to the creator's account, providing participants with
  assurance that the project will not receive all funds immediately.

## ⚠ Platform Dependencies

This RFP is open for proposals. However, full implementation is
blocked until the hard dependencies below are delivered. Proposers
may begin design and development work, but a working on-chain
deployment requires both LP-0015 and an on-chain clock to be
available.

### Hard blockers

#### On-chain clock / timestamp

The tiered sale mechanism is time-driven: each tier becomes active
at its start timestamp and closes at its end timestamp, and the
minimum raise target is evaluated when the final tier closes.
Without a reliable on-chain timestamp, tier activation and close
cannot be enforced. LEZ does not yet have on-chain block time (see
also [RFP-016](./RFP-016-lbp-launchpad.md), which has the same
dependency for LBP weight progression).

#### General cross-program calls (LP-0015)

A buy operation must: (1) call the token program to transfer
collateral from the buyer into the sale account, (2) compute the
token output using the tier's fixed price, (3) call the token program
again to transfer project tokens to the buyer, and (4) update sale
state (collateral raised, tokens remaining, per-wallet allocation
tracking). Without general cross-program calls, the execution cannot
continue after the first token program call, making it impossible to
complete the full buy atomically.

[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this. This prize
is currently **open**.

### Soft blockers

#### Event emission (LP-0012)

Analytics dashboards and sale monitoring services need to react to
sale events (buy executed, tier opened, tier closed, sale entered
refund state). Without structured events, off-chain services must
poll all accounts.

[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

## 👤 Recommended Team Profile

Team experienced with:

- Fixed-price token sale and IDO smart contract design
- Solana or SVM program development (Anchor or native)
- DeFi smart contract security and auditing practices
- Escrow and refund mechanism design
- Front-end development for token sale applications


## ⏱ Timeline Expectations

Estimated duration: **10-12 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Evaluation Criteria

Proposals that meet all hard requirements will be ranked on the
following criteria.

| Criterion | Weight | What we look for |
|-----------|--------|-----------------|
| Technical design quality | 30% | Formal specification of tier state management, refund mechanism, allocation tracking, integer arithmetic strategy, audit plan |
| Privacy architecture | 25% | Strength of anonymity properties in the private account path, completeness of the deshield→buy→re-shield flow, allowlist privacy interaction (if applicable), nullifier-based allocation enforcement |
| Team experience | 20% | Prior token sale, IDO, or escrow protocol work, smart contract security track record, familiarity with SVM or similar execution environments |
| Timeline and milestones | 15% | Realistic schedule with concrete deliverables, risk identification, dependency management (especially LP-0015) |
| Ecosystem alignment | 10% | Open source commitment, composability with other LEZ programs (vesting, DEX), community engagement plan |

## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [RFP-015: Privacy-Preserving Token Launchpad (Bonding Curve)](./RFP-015-bonding-curve-launchpad.md)
  (companion launchpad RFP using the bonding curve mechanism)
- [RFP-016: Privacy-Preserving Token Launchpad (LBP)](./RFP-016-lbp-launchpad.md)
  (companion launchpad RFP using the LBP mechanism)
- [RFP-008: Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (reference for the public/private account interaction pattern used
  across LEZ applications)
- [RFP-017: Privacy-Preserving Token Vesting](./RFP-017-token-vesting.md)
  (soft requirement: post-sale vesting integration)
- [Appendix: Token Launchpad Ecosystem](../appendix/token-launchpad-ecosystem.md)
  (survey of existing launchpad protocols, scale data, refund
  mechanisms, and tiered sale data)

- [LP-0013: Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
- [LP-0014: Token program improvements: ATAs + wallet tooling](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
- [Fjord Foundry tiered sale documentation](https://help.fjordfoundry.com/fjord-foundry-docs/for-sale-creators/seed-private-and-public-rounds)
- [SPEL framework](https://github.com/logos-co/spel)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
