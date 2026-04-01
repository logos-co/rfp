---
id: RFP-015
title: Privacy-Preserving Token Launchpad: Bonding Curve
tier: L
funding: $XXXXX
status: open
dependencies: See Platform Dependencies section
category: Applications & Integrations
---


# RFP-015: Privacy-Preserving Token Launchpad: Bonding Curve

## 🧭 Overview

Build a bonding curve token launchpad on LEZ, using a supply-driven pricing
mechanism where token price increases deterministically as a function of
tokens remaining in the curve. Unlike the time-driven LBP mechanism in
[RFP-016](./RFP-016-lbp-launchpad.md), a bonding curve's price
progression is supply-driven: price advances with each purchase. Participants can buy tokens directly from a public account
or, for privacy, via a deshield→buy→re-shield pattern (see
[RFP-008](./RFP-008-lending-borrowing-protocol.md), which defines this
interaction model for LEZ applications): when a private account is
used, each purchase is routed through a fresh, single-use public
account, hiding the link between the buyer's private identity and their
participation in the sale. An optional private allowlist gate enables
projects to restrict participation without exposing the eligibility set
on-chain.

The bonding curve launchpad complements [RFP-016](./RFP-016-lbp-launchpad.md): where LBPs reward
patient buyers (price falls over time unless buying pressure
counteracts it), bonding curves reward early believers (price rises
with each purchase). The two mechanisms suit different project profiles,
and together they give the Logos ecosystem a complete token launch
toolkit.

## 🔥 Why This Matters

Bonding curves are one of the most proven token distribution mechanisms
in DeFi. pump.fun surpassed $1B in daily trading volume on Solana using a
constant-product bonding curve, demonstrating that the model scales and
attracts broad community participation. Meteora's Dynamic Bonding Curve
is a leading open-source SVM launchpad primitive, used in
production by multiple projects. Beyond Solana, the constant-product
model underpins Uniswap, Balancer, and virtually every major AMM; the
formula is the most audited and best-understood pricing primitive in
DeFi.

## 🏗 Design Rationale

### Constant product AMM with virtual reserves

The recommended pricing mechanism is the **constant product AMM with
virtual reserves**, as used by pump.fun and Meteora's Dynamic Bonding
Curve. It is preferred over alternatives (polynomial integral curves,
Bancor power function) for three reasons: it is proven at scale on
Solana; the implementation requires only multiplication and division,
with no fixed-point exponentiation or approximation, making it
straightforward to audit; and correctness reduces to a single invariant
that any reviewer can check. The full formula specification is in the
Reference Implementation section. Teams may propose alternative
mechanisms under the deviation standard described there.

### Supply-driven close over time-driven close

The bonding curve's canonical close condition is a **supply target**:
the sale ends when all `D` sale tokens have been purchased. A supply
target defines the sale's complete economic trajectory at creation time:
the starting price, the price at any given supply point, and the
total collateral raised at graduation are all computable before the
sale opens. This gives buyers a calculable price ceiling and gives the
project a predictable raise range.

A time-based close creates a weaker invariant: if buying stops early,
the sale ends with unsold supply and no clear clearing price.
Supply-based close avoids this. Time-based end is retained as a soft
requirement for projects that need a hard deadline.

### One-directional sale

The sale is one-directional: buyers purchase tokens from the curve;
there is no sell-back path during the sale. This is the correct choice
for a token launch: it prevents reflexive sell pressure during the
distribution window, simplifies pool accounting, and removes the attack
surface of circular buy-sell cycles. The accumulated collateral reserve
is locked until close and either withdrawn by the creator or used to
seed a DEX pool (see auto-graduation in Soft Requirements).

Optional sell-back (two-way curve) is a soft requirement for projects
that want continuous liquidity from day one rather than a bounded
sale-plus-DEX-seed model.

### Public pool state

All bonding curve state is public on-chain: virtual reserve values,
invariant `k`, real reserve balances, current spot price,
open/closed status. As with [RFP-016](./RFP-016-lbp-launchpad.md),
this is a deliberate architectural choice. Public state enables
permissionless price verification (any participant can verify
`tokens_out` against the formula independently), composability with
other LEZ programs, and verifiable sale analytics without
cryptographic complexity in the curve program itself. Participant
privacy is enforced at the UX layer via the optional
deshield→buy→re-shield pattern, which does not require private pool
state.

### Optional allowlist

The allowlist gate is optional: many projects want permissionless open
sales. When gating is needed (geographic restrictions, community-only
rounds, pre-selected investor lists), the creator can commit an
eligibility set at creation time and restrict buys to participants who
can prove inclusion. The implementation approach is left to the
proposing team.

### Bonding curves and LBPs as complementary mechanisms

The Logos ecosystem provides both a bonding curve launchpad (this RFP)
and an LBP launchpad ([RFP-016](./RFP-016-lbp-launchpad.md)). The two
mechanisms are not redundant: they suit different project profiles and
together provide a complete token launch toolkit.

A bonding curve rewards early participation with a lower entry price
and provides a fully deterministic, supply-driven price trajectory. The
complete economic outcome is computable before the sale opens: starting
price, price at any supply point, and total collateral at graduation
are all known in advance. Graduation to a DEX pool is natural, since
the virtual reserve model accumulates both token and collateral reserves
in the correct ratio throughout the sale. This suits projects that want
to reward early believers, need a transparent and auditable price
formula, and value a predictable raise range.

An LBP opens above estimated fair value and lets the market discover
price over multiple days, without requiring the project to pre-set a
valuation. It is better suited to projects that want broad, patient
participation and market-driven price discovery. The two mechanisms
are genuinely different tools for genuinely different launch
strategies: teams should choose based on their community profile and
distribution goals.

### Known limitations of bonding curves for token launches

Bonding curves have weaker bot deterrence than LBPs. In an LBP, buying
early is unprofitable because price is highest at the start and falls
over time. In a bonding curve, buying early is cheapest: bots that
execute at the first block of an open sale acquire tokens at the
starting spot price `p₀ = Vc / Vt`, before the broader community can
participate. The allowlist gate (restricted early access) and
per-transaction buy limits (see Functionality requirement 2) are
available mitigations, but neither eliminates the advantage entirely.
Projects seeking the strongest bot-deterrence properties should use
RFP-016 instead.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a bonding curve program on LEZ with a deterministic,
   supply-driven pricing mechanism that maintains a well-defined
   invariant across all buy operations. The buy instruction accepts
   a collateral input `C_in` and computes a deterministic token
   output based on the current curve state. The SDK must also expose
   the inverse: the exact collateral cost for a buyer who requests a
   specific token quantity `Q`. After each buy, the curve state must
   be updated to preserve the pricing invariant. The real token
   reserve decreases by `tokens_out`; the real collateral reserve
   increases by `C_in`. If the computed `tokens_out` would exceed the
   remaining real token reserve, the transaction must revert. The
   pricing invariant must never change after creation. See the
   Reference Implementation section for the recommended formula and
   the deviation standard for alternative mechanisms.
2. A sale creator can configure a sale with the following parameters:
   - Token pair (project token + collateral token).
   - Virtual token reserves `Vt`: the total project token supply
     deposited by the creator into the curve program at creation.
     All `Vt` tokens must be transferred from the creator at sale
     creation.
   - Virtual collateral reserves `Vc`: a synthetic starting value;
     no real collateral is deposited by the creator. Together with
     `Vt`, determines `k = Vt × Vc` (computed and stored at
     creation) and the starting spot price `p₀ = Vc / Vt`.
   - Sale quantity `D`: the number of tokens made available for
     purchase (`D < Vt`). The remaining `Vt − D` tokens are held
     in reserve within the program for post-graduation DEX seeding.
     `D` is also the supply target: the sale auto-closes when `D`
     tokens have been sold.
   - Optional: minimum raise threshold (see item 6 below).
   - Optional: per-transaction buy limit (maximum collateral amount
     spendable in a single buy transaction).
   - Optional: private allowlist gate (see item 8 below).
3. Participants buy project tokens from the curve using either a
   public account directly, or via the deshield→buy→re-shield
   pattern for private account interaction (see
   [RFP-008](./RFP-008-lending-borrowing-protocol.md), which defines
   this interaction model for LEZ applications). Both paths must be
   supported by the program and SDK.
4. The sale closes automatically when the real token reserve reaches
   zero (supply target `D` reached). The creator can also close the
   sale manually at any time before the target is reached.
5. After the sale closes, the creator can withdraw:
   - The real collateral raised (net of fees).
   - The reserved `Vt − D` tokens (if not used for auto-graduation).
6. The sale creator may configure an optional minimum raise threshold
   at creation time. If the creator closes the sale manually before
   the supply target is reached, and the real collateral reserve is
   below this threshold, the sale enters a **refund state**: no
   further withdrawals are permitted by the creator, and buyers may
   submit a refund claim to recover their collateral (see Reliability
   requirement 4). For buyers who used the private account path,
   the refund is credited to the ephemeral public account used during
   the buy; the SDK must persist the ephemeral account keypair, or
   delegate that persistence to the wallet module, so the buyer can
   submit the refund claim. If the supply target is reached
   automatically (full sell-through), the minimum raise check is
   skipped; a complete sell-through always succeeds.
7. Slippage protection: buyers specify the collateral amount to spend
   and a minimum token quantity they are willing to accept. The
   transaction reverts if the computed `tokens_out` is below this
   minimum.
8. The sale creator can enable an optional allowlist gate. When
   enabled, only participants who can prove inclusion in the
   committed eligibility set may buy from the curve. The proposing
   team must specify and justify their allowlist mechanism in their
   application.
9. The sale creator can pause buying at any time during the sale
   (emergency stop). Pausing does not affect the curve state, the
   virtual reserves, or the invariant `k`.
10. Use Associated Token Accounts (ATAs) for all token interactions,
    consistent with
    [LP-0014](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
    and [RFP-008](./RFP-008-lending-borrowing-protocol.md).

#### Usability

1. Provide an SDK for building Logos modules that interact with the
   bonding curve program. The SDK must expose the full lifecycle for
   both participants (discover active sales, compute price and
   impact, buy, query position) and creators (create sale,
   pause/resume, close, withdraw). The SDK must support both direct
   public account interaction and the deshield→buy→re-shield pattern
   for private account interaction. When the private account path is
   used, the SDK must handle the atomic deshield (both collateral
   and gas) as a single indivisible user action.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo. The mini-app must cover:
   - **Participant view**: browse active sales with current spot
     price, real token reserve as a percentage of the supply
     target `D`, total collateral raised, and a price-vs-supply
     chart; execute a buy; view purchase history.
   - **Creator view**: create a new sale (all parameters including
     `Vt`, `Vc`, sale quantity `D`, allowlist configuration),
     monitor an active sale (live supply progress, collateral
     raised), pause/resume, close sale, and withdraw proceeds.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations for both participants (buy,
   query price, check sale status) and creators (create sale,
   pause/resume, close, withdraw).
4. The mini-app must display a pre-buy confirmation summary before
   each purchase: collateral to spend, exact tokens to be received
   (computed using the pricing formula), current spot price
   (`Vc / Vt`), price impact (percentage increase in spot price
   after the buy), and fee.
5. When using the private account path, the mini-app must display a
   privacy disclosure before each buy, identifying what will be
   visible on-chain (buy transaction, collateral amount, tokens
   received, curve address, ephemeral account) and what will not be
   traceable (the buyer's private account and the subsequent
   destination of purchased tokens).
6. When using the private account path, the mini-app must enforce
   the atomic deshield, preventing a buyer from funding the
   ephemeral account from any external source, which would create
   an on-chain link to an existing identity.
7. When using the private account path, the mini-app must confirm
   before each buy that the buyer's shielded balance covers both
   the collateral cost and the gas fee within the single deshield
   action. A clear, actionable error must be shown if the balance
   is insufficient.
8. Provide a sale analytics view showing, for each active or
   completed sale: total collateral raised, current spot price,
   supply sold over time (progress chart), price-vs-supply curve
   with current position marked, and number of buy transactions.
   Analytics must not expose individual participant identities or
   link buy transactions to specific accounts.
9. Provide an IDL for the bonding curve program using the
   [SPEL framework](https://github.com/logos-co/spel).
10. Failed or rejected buys must return clear, actionable error
    messages (e.g., insufficient balance, supply target already
    reached, sale paused, allowlist gate rejected, slippage
    exceeded, per-transaction buy limit exceeded).

#### Reliability

1. Curve state (`Vt`, `Vc`, `k`, real reserves) must remain
   consistent under concurrent buy submissions. No double-spend,
   incorrect accounting, or invariant violation.
2. A failed buy must revert atomically: the buyer's collateral is
   not consumed and the curve state is unchanged.
3. Auto-close on supply target: when the final sale tokens are sold
   and the real token reserve reaches zero, the sale must close
   atomically in the same transaction. No additional close
   instruction should be required; no further buys must be accepted
   after close.
4. If a sale enters refund state (minimum raise not met after manual
   close), the program must maintain per-buyer purchase records for
   all public-account purchases, including ephemeral accounts used
   in private account buys, enabling each buyer to claim their
   collateral refund independently. Refund claims must be
   idempotent: a buyer cannot claim the same purchase twice.
5. The creator cannot withdraw collateral while the sale is in
   refund state.

#### Performance

1. A single buy transaction completes within one LEZ transaction.
2. A close transaction (manual or auto-triggered by final buy)
   completes within one LEZ transaction.
3. Document the compute unit (CU) cost of each operation: create
   sale, buy, close sale, withdraw, refund claim (if applicable).
   Note the LEZ testnet version against which measurements were
   taken.

#### Supportability

Proposals must include separate milestones for testnet 0.2, testnet 0.3,
and mainnet deployment.

1. The bonding curve program is deployed and tested on LEZ testnet 0.2.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI. CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test. Test
   coverage must include: invariant preservation across multiple
   buys, happy-path buy, slippage revert (tokens_out below
   minimum), allowlist gate accept and reject, auto-close on supply
   target, manual close with minimum raise met, manual close with
   minimum raise not met (refund path), refund claim by buyer,
   pause/resume, per-transaction buy limit enforcement.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for both creators and
   participants via CLI and mini-app.
5. The program is updated and verified on LEZ testnet 0.3.
6. The program is deployed to LEZ mainnet.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→buy→re-shield pattern for private
   account interaction. When a user chooses the private account
   path, the SDK must enforce the complete pattern; the re-shield
   step must not be skippable.
2. When using the private account path, the mini-app must display
   the pre-confirmation privacy summary described in the Usability
   section before each buy operation.
3. When using the private account path, the SDK must validate that
   the re-shield target for purchased tokens is a private (shielded)
   account before submitting the transaction, and reject with an
   explicit error if it is not.
4. The ephemeral public account created during the deshield step
   must never be reused across operations. Each buy from a private
   account must use a freshly generated account with no prior
   on-chain history.

### Privacy Architecture

All bonding curve state is public on-chain (virtual reserve values
`Vt` and `Vc`, invariant `k`, real reserve balances, current spot
price, open/closed status). This is a deliberate architectural
choice: public state enables permissionless price verification (any
participant can verify `tokens_out` against the pricing formula
independently), composability, and sale analytics without
cryptographic complexity in the curve program itself.

User privacy is optionally enforced at the UX layer. The mini-app and
SDK support both direct public account interaction and private account
interaction via the deshield→buy→re-shield pattern. When users opt to
interact from a private account, the SDK must enforce the complete
pattern as described below.

#### Interaction flow

For every buy from a private account:

1. The buyer initiates from their private account. The SDK deshields
   to a **fresh, single-use** public account (account A) with no
   prior on-chain history. The deshield atomically transfers both
   the collateral token and enough native token for gas in a single
   indivisible action.
2. Account A executes the buy against the bonding curve program.
3. Account A re-shields the purchased project tokens to the buyer's
   private account. Account A is never reused.

> **Gas:** Both collateral and gas must come exclusively from the
> deshield in step 1. Funding account A from any external source
> (such as a CEX withdrawal or a known wallet) creates an
> on-chain link to an existing identity and breaks the privacy
> guarantee. The SDK must make this impossible; the atomic deshield
> is a single, indivisible user action.

#### What is public (observable on-chain)

- All curve state: token pair, virtual reserves (`Vt`, `Vc`),
  invariant `k`, real token reserve, real collateral reserve,
  sale quantity `D`, current spot price, open/closed status.
- All buy transactions: collateral spent, tokens received, and
  block height. When using the private account path, the buyer's
  address is an ephemeral intermediary account with no prior
  on-chain history.
- Allowlist gate configuration and whether the gate is enabled,
  but not the list of eligible addresses.
- Sale close and creator withdrawal transactions.

#### What is private (when using the private account path)

- Which private account originated the collateral for a buy.
- Where purchased tokens go after re-shielding.
- Any link between multiple buys by the same buyer (no on-chain
  linkability across ephemeral accounts).
- Whether a specific private account participated in the sale at all.

### Known Limitations

**Per-wallet buy limits are not supported for the private account
path.** Enforcing a cap on cumulative spend per participant requires
tracking spend per address. For public account interactions this is
feasible, but is incompatible with the private account path: each buy
from a private account uses a fresh ephemeral public address with no
prior history, so the program cannot link multiple buys to the same
underlying participant. The per-transaction buy limit (Functionality
requirement 2) is a partial mitigation: it bounds the collateral
spendable in a single buy, which raises the cost of rapid accumulation,
but it does not prevent a single participant from submitting many
transactions. Projects that need strong concentration limits should
use the allowlist gate to restrict the eligible set.

**Refunds on the private account path require local keypair retention.**
If a sale enters refund state, the refund is credited to the ephemeral
public account used during the original buy. The buyer can reclaim their
collateral by signing the refund claim with that ephemeral account's
keypair. The SDK must persist all ephemeral keypairs used in buy
transactions, or delegate that persistence to the wallet module. If the keypair is lost, the refund
cannot be claimed. The mini-app must clearly communicate this dependency
when a minimum raise is configured and the buyer is using the private
account path.

**Front-running at sale open.** Unlike the LBP mechanism, which opens
at a price above estimated fair value and declines over time, a bonding
curve opens at its lowest price (`p₀ = Vc / Vt`). Bots that execute
at the first block of the sale acquire tokens at the cheapest possible
price, before the community can participate. The allowlist gate and
per-transaction buy limit reduce the impact but do not eliminate it.
Projects requiring stronger bot deterrence should use
[RFP-016](./RFP-016-lbp-launchpad.md) instead.

### Reference Implementation

The recommended pricing mechanism is the **constant product AMM with
virtual reserves**, as described in the Design Rationale. Proposals
that use this formula require no additional justification beyond the
hard requirements above.

The constant product invariant is `Vt × Vc = k`, where `Vt` is the
virtual token reserve and `Vc` is the virtual collateral reserve. The
buy formula computes token output as:

```
tokens_out = Vt - k / (Vc + C_in)
```

The inverse (exact collateral cost for a requested token quantity `Q`):

```
C_in = k / (Vt - Q) - Vc
```

After each buy, `Vt` and `Vc` are updated to preserve `k`. `k` is
computed at creation as `Vt × Vc` and must never change.

**Deviation standard.** Teams may propose an alternative pricing
mechanism (such as a polynomial integral, the Bancor power function,
a piecewise constant product as used in Meteora DBC, a batch auction,
or a commit-reveal scheme) provided the proposal includes: (1) a
formal specification equivalent in detail to the formulas above,
(2) a security argument that the mechanism's core invariant is
preserved under all operations, and (3) citations to existing
production deployments or audits.

### Soft Requirements

- **Two-way (sell-back) curve**: after purchasing tokens, holders
  can sell them back to the curve, receiving collateral from the
  real reserve. The sell formula is the inverse of the buy formula:
  `C_out = Vc - k / (Vt + tokens_in)`. This transforms the bonding
  curve from a one-time sale vehicle into a continuous liquidity
  source, but introduces reflexive sell pressure during the sale
  window; proposals including this feature must address the
  implications for the minimum raise mechanism.
- **Auto-graduation to DEX**: when the supply target is reached and
  the sale auto-closes, the program can automatically deploy the
  accumulated real collateral reserve and the reserved `Vt − D`
  tokens as liquidity into a LEZ DEX pool (requires
  [RFP-004](./RFP-004-privacy-preserving-dex.md) and LP-0015 to
  be available). This eliminates manual post-sale liquidity seeding
  and provides immediate post-graduation tradability.
- **Time-based sale end**: support an optional end timestamp in
  addition to the supply target, closing the sale at whichever
  condition is reached first.
- **Integration with token vesting**: at sale close, purchased
  tokens can be routed directly into a vesting schedule (see
  [RFP-017](./RFP-017-token-vesting.md)) rather than transferred
  to the buyer immediately. This enables a sale-plus-vesting flow
  in a single operation.

## ⚠ Platform Dependencies

This RFP is open for proposals. However, full implementation is
blocked until the hard dependency below is delivered. Proposers may
begin design and development work, but a working on-chain deployment
requires LP-0015 to be available.

### Hard blockers

#### General cross-program calls (LP-0015)

A buy operation must: (1) call the token program to transfer
collateral from the buyer into the real collateral reserve, (2)
compute `tokens_out` using the constant product formula, (3) call
the token program again to transfer project tokens to the buyer, and
(4) update curve state (`Vt`, `Vc`, real reserves). Without general
cross-program calls, the execution cannot continue after the first
token program call, making it impossible to complete the full buy
atomically.

[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this. This prize
is currently **open**.

### Soft blockers

#### Event emission (LP-0012)

Analytics dashboards and monitoring services need to react to curve
events (buy executed, sale closed, refund claimed). Without structured
events, off-chain services must poll all accounts.

[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

## 👤 Recommended Team Profile

Team experienced with:

- AMM and constant product pool design
- Solana or SVM program development (Anchor or native)
- DeFi smart contract security and auditing practices
- Fixed-point arithmetic and precision-safe integer math for on-chain AMMs
- Front-end development for token sale and trading applications


## ⏱ Timeline Expectations

Estimated duration: **10–12 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [RFP-016: Privacy-Preserving Token Launchpad (LBP)](./RFP-016-lbp-launchpad.md)
  (companion launchpad RFP using the LBP mechanism)
- [RFP-008: Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (reference for the public/private account interaction pattern used
  across LEZ applications)
- [RFP-017: Privacy-Preserving Token Vesting](./RFP-017-token-vesting.md)
  (soft requirement: post-sale vesting integration)
- [RFP-004: Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (soft requirement: auto-graduation target)

- [LP-0013: Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
- [LP-0014: Token program improvements: ATAs + wallet tooling](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
- [Meteora Dynamic Bonding Curve (program and docs)](https://github.com/MeteoraAg/dynamic-bonding-curve)
  (primary reference implementation for constant product bonding curve on Solana)
- [pump.fun bonding curve mechanism](https://deepwiki.com/pump-fun/pump-public-docs/3.1-pump-bonding-curve-mechanism)
  (canonical production deployment of constant product with virtual reserves)
- [pumpdotfun-sdk: bondingCurveAccount.ts](https://github.com/rckprtr/pumpdotfun-sdk/blob/main/src/bondingCurveAccount.ts)
  (TypeScript reference for buy/sell formula)
- [SPEL framework](https://github.com/logos-co/spel)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
