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
participation in the sale.

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
Supply-based close avoids this. No bonding curve platform in the
ecosystem implements time-based close; all use supply or price
targets exclusively (see the
[Sale Lifecycle](../appendix/token-launchpad-ecosystem.md#sale-lifecycle-and-close-mechanics)
section of the appendix). An optional end timestamp is available as
a soft requirement for projects that need a hard deadline.

### Two-way curve

The bonding curve supports both buy and sell operations, matching
the ecosystem standard established by Pump.fun. Participants can
buy tokens from the curve and sell them back at the current AMM
price. The sell formula is the inverse of the buy formula (see
Reference Implementation). This provides continuous liquidity
during the sale window: participants who want to exit can do so
at market price without waiting for graduation and DEX listing.

The two-way design introduces reflexive sell pressure during the
sale: if early buyers sell back into the curve, the price drops,
potentially discouraging new participants. Projects that want to
prevent this dynamic can use the optional one-directional mode
(see Soft Requirements), which disables selling during the sale
window.

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
participate. No bonding curve platform in the ecosystem implements
access restrictions or bot mitigation at the protocol level.
Projects seeking stronger bot-deterrence properties should use
RFP-016 instead.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement a bonding curve program on LEZ with a deterministic,
   supply-driven pricing mechanism that maintains a well-defined
   invariant across all buy and sell operations. The buy instruction
   accepts a collateral input `C_in` and computes a deterministic
   token output based on the current curve state. The SDK must also
   expose the inverse: the exact collateral cost for a buyer who
   requests a specific token quantity `Q`. The sell instruction
   accepts a token input `tokens_in` and computes a deterministic
   collateral output from the real collateral reserve. Sell
   transactions must not withdraw more collateral than the real
   reserve holds. After each buy, the real token reserve decreases
   by `tokens_out` and the real collateral reserve increases by
   `C_in`. After each sell, the real token reserve increases by
   `tokens_in` and the real collateral reserve decreases by
   `C_out`. If the computed `tokens_out` on a buy would exceed the
   remaining sale reserve, the transaction must revert. All
   arithmetic must use integer-only operations and round against
   the trader: on buy, `tokens_out` rounds down and `C_in` rounds
   up; on sell, `C_out` rounds down. This ensures the pool remains
   solvent and the pricing invariant is never violated by rounding.
   The pricing invariant must never change after creation. See the
   Reference Implementation section for the recommended formulas
   and the deviation standard for alternative mechanisms.
2. A sale creator can configure a sale with the following parameters:
   1. Token pair (project token + collateral token).
   2. Sale quantity `D`: the number of tokens available for
      purchase. All `D` tokens must be transferred from the creator
      at sale creation.
   3. Optional: DEX seed quantity `R`: tokens reserved for
      post-graduation DEX seeding (may be zero). If set, `R`
      tokens must also be transferred from the creator at sale
      creation. Total real deposit is `D + R`.
   4. Virtual token reserve `Vt`: a synthetic pricing parameter
      that determines the shape of the bonding curve (`Vt > D`).
      This is not the deposit amount; it is typically larger than
      `D + R` to position the starting price on the curve.
   5. Virtual collateral reserve `Vc`: a synthetic starting value;
      no real collateral is deposited by the creator. Together
      with `Vt`, determines `k = Vt × Vc` (computed and stored at
      creation) and the starting spot price `p₀ = Vc / Vt`.

   The program must maintain two distinct accounting buckets: a
   **sale reserve** (starts at `D`, decreases with each purchase)
   and a **DEX seed reserve** (starts at `R`, untouched until
   close). `D` is the supply target: the sale auto-closes when
   the sale reserve is exhausted.
3. Participants buy and sell project tokens on the curve using
   either a public account directly, or via the
   deshield→trade→re-shield pattern for private account interaction
   (see [RFP-008](./RFP-008-lending-borrowing-protocol.md), which
   defines this interaction model for LEZ applications). Both paths
   must be supported by the program and SDK.
4. The sale closes automatically when the sale reserve is exhausted
   (all `D` tokens have been sold).
5. After the sale closes, the creator can withdraw:
   - The real collateral raised, net of fees as defined by the
     fee model specified in the proposal (see the Fee structure
     subsection in Design Rationale).
   - The DEX seed reserve `R` tokens (if not used for
     auto-graduation).
6. Slippage protection: on buy, buyers specify the collateral
   amount to spend and a minimum token quantity they are willing to
   accept; the transaction reverts if the computed `tokens_out` is
   below this minimum. On sell, sellers specify the token quantity
   to sell and a minimum collateral amount they are willing to
   accept; the transaction reverts if the computed `C_out` is below
   this minimum.
7. Use Associated Token Accounts (ATAs) for all token interactions,
    consistent with
    [LP-0014](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
    and [RFP-008](./RFP-008-lending-borrowing-protocol.md).

#### Usability

1. Provide an SDK for building Logos modules that interact with the
   bonding curve program. The SDK must expose the full lifecycle for
   both participants (discover active sales, compute price and
   impact, buy, query position) and creators (create sale,
   close, withdraw). The SDK must support both direct
   public account interaction and the deshield→buy→re-shield pattern
   for private account interaction. When the private account path is
   used, the SDK must handle the atomic deshield (both collateral
   and gas) as a single indivisible user action.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo. The mini-app must cover:
   - **Participant view**: browse active sales with current spot
     price, sale reserve as a percentage of the supply
     target `D`, total collateral raised, and a price-vs-supply
     chart; execute a buy; view purchase history.
   - **Creator view**: create a new sale (all parameters including
     `Vt`, `Vc`, sale quantity `D`),
     monitor an active sale (live supply progress, collateral
     raised), close sale, and withdraw proceeds.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations for both participants (buy,
   query price, check sale status) and creators (create sale,
   close, withdraw).
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
    reached, slippage
    exceeded).
#### Reliability

1. Curve state (`Vt`, `Vc`, `k`, real reserves) must remain
   consistent under concurrent buy submissions. No double-spend,
   incorrect accounting, or invariant violation.
2. A failed buy must revert atomically: the buyer's collateral is
   not consumed and the curve state is unchanged.
3. Auto-close on supply target: when the final sale tokens are sold
   and the sale reserve is exhausted, the sale must close
   atomically in the same transaction. No additional close
   instruction should be required; no further buys must be accepted
   after close.

#### Performance

1. A single buy transaction completes within one LEZ transaction.
2. A close transaction (manual or auto-triggered by final buy)
   completes within one LEZ transaction.
3. Document the compute unit (CU) cost of each operation: create
   sale, buy, close sale, withdraw.
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
   minimum), auto-close on supply
   target, manual close.
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
  invariant `k`, sale reserve, DEX seed reserve, real collateral
  reserve, sale quantity `D`, current spot price, open/closed
  status.
- All buy transactions: collateral spent, tokens received, and
  block height. When using the private account path, the buyer's
  address is an ephemeral intermediary account with no prior
  on-chain history.
- Sale close and creator withdrawal transactions.

#### What is private (when using the private account path)

- Which private account originated the collateral for a buy.
- Where purchased tokens go after re-shielding.
- Any link between multiple buys by the same buyer (no on-chain
  linkability across ephemeral accounts).
- Whether a specific private account participated in the sale at all.

#### Trust assumptions

The privacy guarantees above depend on the buyer using an SDK or
client that correctly implements the full deshield→buy→re-shield
pattern. The bonding curve program itself cannot enforce the
re-shield step: a buyer who calls the program directly or uses a
non-conforming client could skip the re-shield, leaving purchased
tokens in the ephemeral public account and breaking their own
anonymity. The program enforces correctness of the buy (invariant
preservation, slippage checks); the privacy pattern is enforced
by the SDK, not by the on-chain program. Buyers who bypass the
SDK accept full responsibility for any resulting privacy loss.

### Known Limitations

**Front-running at sale open.** Unlike the LBP mechanism, which opens
at a price above estimated fair value and declines over time, a bonding
curve opens at its lowest price (`p₀ = Vc / Vt`). Bots that execute
at the first block of the sale acquire tokens at the cheapest possible
price, before the community can participate. No bonding curve platform
in the ecosystem implements access restrictions or bot mitigation at
the protocol level. Projects requiring stronger bot deterrence should
use [RFP-016](./RFP-016-lbp-launchpad.md) instead.

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

The sell formula computes collateral output as:

```
C_out = Vc - k / (Vt + tokens_in)
```

After each buy, `Vt` decreases by `tokens_out` and `Vc` increases
by `C_in`. After each sell, `Vt` increases by `tokens_in` and `Vc`
decreases by `C_out`. In both cases, `k = Vt × Vc` is preserved.
`k` is computed at creation and must never change.

**Deviation standard.** Teams may propose an alternative pricing
mechanism (such as a polynomial integral, the Bancor power function,
a piecewise constant product as used in Meteora DBC, a batch auction,
or a commit-reveal scheme) provided the proposal includes: (1) a
formal specification equivalent in detail to the formulas above,
(2) a security argument that the mechanism's core invariant is
preserved under all operations, and (3) citations to existing
production deployments or audits.

### Soft Requirements

- **One-directional mode**: a creator-configurable option that
  disables sell-back during the sale window. When enabled, buyers
  can only purchase tokens from the curve; the accumulated
  collateral reserve is locked until close. This prevents reflexive
  sell pressure during the distribution window and simplifies pool
  accounting, at the cost of removing exit liquidity for
  participants before graduation.
- **Auto-graduation to DEX**: when the supply target is reached and
  the sale auto-closes, the program can automatically deploy the
  accumulated real collateral reserve and the DEX seed reserve `R`
  tokens as liquidity into a LEZ DEX pool (requires
  [RFP-004](./RFP-004-privacy-preserving-dex.md) and LP-0015 to
  be available). This eliminates manual post-sale liquidity seeding
  and provides immediate post-graduation tradability.
- **Optional end timestamp**: the sale creator can configure an end
  timestamp at creation time. The sale closes when the supply target
  is reached or the end timestamp passes, whichever comes first.
  This prevents zombie sales (curves that sit open indefinitely if
  demand is low). On Pump.fun, the graduation rate is approximately
  0.7% to 1.4%, meaning over 98% of bonding curves never reach
  their supply target and remain as dormant on-chain state. When an
  end timestamp is configured, the program must enforce a minimum
  sale duration at creation time, long enough that latency in the
  deshield→buy→re-shield privacy path does not systematically
  disadvantage private-path buyers on price relative to public-path
  buyers. No bonding curve platform currently implements this
  feature.


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
events (buy executed, sale closed). Without structured
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
- [RFP-004: Privacy-Preserving DEX](./RFP-004-privacy-preserving-dex.md)
  (soft requirement: auto-graduation target)
- [Appendix: Token Launchpad Ecosystem](../appendix/token-launchpad-ecosystem.md)
  (survey of existing launchpad protocols, scale data, and refund mechanisms)

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
