---
id: RFP-006
title: Privacy-Preserving Token Launchpad
tier: XL
funding: $XXXXX
status: draft
category: Applications & Integrations
---


# RFP-006 — Privacy-Preserving Token Launchpad

## 🧭 Overview

Build a token launchpad on LEZ using a Liquidity Bootstrapping Pool
(LBP) mechanism — a time-limited, weight-shifting AMM that enables
fair, bot-resistant price discovery for new token launches.
Participants can buy tokens directly from a public account or, for
privacy, via a deshield→buy→re-shield pattern (see
[RFP-008](./RFP-008-lending-borrowing-protocol.md)): when a private
account is used, each purchase is routed through a fresh, single-use
public account, hiding the link between the buyer's private identity
and their participation in the sale. An optional private allowlist
gate enables projects to restrict participation without exposing the
eligibility set on-chain.

The launchpad is the entry point for new projects into the Logos
ecosystem. It is where token price and initial distribution are
established. The team building this should have strong experience in
AMM or order-book design, SVM program development, and DeFi
application security. Familiarity with weighted AMM pool mechanics
(Balancer-style) and the Logos privacy interaction model is strongly
recommended.

## 🔥 Why This Matters

LBPs have become the dominant mechanism for fair, permissionless token
price discovery. Fjord Foundry's own FJO token LBP raised $15.35M in
April 2024 ([DL News](https://www.dlnews.com/articles/defi/fjord-foundry-raises-15-million-in-fjo-token-sale/)),
described at the time as the highest LBP raise of the year. Autonolas
raised $547k from $50k in initial USDC liquidity via an LBP on the
same platform
([olas.network](https://olas.network/blog/olas-public-launch)).
The mechanism deters bots — price starts above fair value and falls
unless buying pressure counteracts it — and removes the need for
project teams to pre-set a valuation for their token.

On transparent chains, participation in a token sale is immediately
linkable to a wallet address and, by extension, to the buyer's full
transaction history. Early investors and community members can be
identified, targeted for social engineering, or front-run at the
unlock date. Platforms like Fjord Foundry, DAO Maker, and Polkastarter
all require participants to expose their wallet identities on-chain;
those using KYC leak that data to a centralised service.

On LEZ, the optional deshield→buy→re-shield pattern means a buyer's
private account is never linked to their purchase on-chain. Observers
see a buy transaction from an ephemeral public account with no prior
history. The buyer's identity, their total position size, and the
destination of their purchased tokens are all private. This is a
structurally different — and stronger — privacy guarantee than
anything available on existing launchpad platforms.

## 🏗 Design Rationale

### LBP over fixed-price, Dutch auction, or sealed-bid auction

Three alternative mechanisms exist for token sales: fixed-price
(set a price upfront, sell until allocation runs out), Dutch auction
(price declines on a curve until all tokens clear), and sealed-bid
auction
([LP-0004](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0004.md)).
LBP is chosen as the primary mechanism because it is the only one
that does not require the project to pre-set a valuation.

Fixed-price forces a binary gamble: underpricing benefits buyers and
dilutes the project; overpricing leads to an incomplete raise and
reputation damage. Dutch auctions clear at a single price moment,
which creates a coordination game — buyers wait for the price to fall
and then rush simultaneously, producing congestion and poor UX.
LP-0004 (sealed-bid) hides individual bids and is well-suited to
single-item auctions, but clears at one moment rather than over days,
making it unsuitable for broad community distribution.

LBP addresses these problems by running price discovery continuously
over days: the token price starts above estimated fair value and falls
unless buying pressure counteracts it. Bots are naturally deterred —
early sniping is unprofitable because price is highest at the start.
The mechanism has a strong empirical track record (Fjord Foundry,
Copper, Balancer) and is well understood by the DeFi builder
community.

Fixed-price mode is retained as a soft requirement for projects that
have an existing community with a known clearing price and prefer
simplicity over discovery.

### Public pool state

A fully private AMM pool — where token reserves and price are
shielded — would provide stronger privacy guarantees, but it is an
open research problem with no practical implementation on any
network. It would also undermine the LBP mechanism itself: price
discovery requires all participants to observe the same current price
curve so they can make informed decisions about when to buy.

Public pool state is the correct choice here. It enables
permissionless price transparency (every participant sees the same
price), composability with other LEZ programs (e.g., routing,
aggregators), permissionless weight poke transactions (anyone can
advance the schedule without depending on the creator), and
verifiable sale analytics. Participant privacy is instead enforced
at the UX layer via the optional deshield→buy→re-shield pattern,
which unlinks the buyer's private identity from their on-chain
purchase without requiring private pool state.

### Optional allowlist and two-tier gating

The allowlist gate is optional because many projects want
permissionless open sales — adding a mandatory gate would make the
protocol less generally applicable. When projects do need access
control (e.g., geographic restrictions, community-only sales,
pre-selected investor lists), the gate is available.

Two gating modes are specified because of the dependency on LP-0003.
The ZK-based gate is preferred: a Merkle proof reveals the claimer's
account ID at submission time, linking that address to their
participation history. A ZK membership proof avoids this entirely.
However, LP-0003 is a draft prize that may not be complete when this
RFP is being built. The Merkle-proof fallback ensures the gating
feature is usable immediately, with an upgrade path to full privacy
when LP-0003 ships.

### Sale pause capability

The pause function is an emergency stop for security incidents —
analogous to the freeze authority in
[RFP-002](./RFP-002-freeze-authority-poc.md). Crucially, pausing does
not halt weight progression: the weight schedule continues to advance
during a pause. This prevents a creator from gaming the mechanism by
pausing at a weight that is artificially favourable and resuming once
they detect large buy orders.

### Minimum raise as an optional parameter

The minimum raise threshold protects buyers: if a sale fails to reach
the floor, all collateral is returned and the project does not receive
funds from a failed launch. This mirrors standard crowdfunding
protections and prevents projects from extracting liquidity from
low-conviction raises.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement an LBP program on LEZ. An LBP is a two-asset pool
   consisting of a project token and a collateral token (e.g.,
   stablecoin or native token), with start and end weights configured
   by the sale creator. Pool weights shift linearly from the start
   weight to the end weight over the sale duration, causing the token
   price to decline over time unless buying pressure counteracts it.
2. A sale creator can configure a sale with the following parameters:
   - Token pair (project token + collateral token).
   - Start and end weights (e.g., 99/1 → 1/99 for the
     token/collateral ratio).
   - Sale start and end timestamps.
   - Initial token deposit amount.
   - Optional: minimum raise (sale is considered failed and
     collateral is refunded to the creator if the minimum is not
     reached by close).
   - Optional: private allowlist gate (see item 7 below).
3. Participants buy project tokens from the pool using either a
   public account directly, or via the deshield→buy→re-shield
   pattern for private account interaction (see
   [RFP-008](./RFP-008-lending-borrowing-protocol.md)). Both paths
   must be supported by the program and SDK.
4. Pool weights shift deterministically according to the configured
   schedule. Any account can submit a weight-update ("poke")
   transaction to advance the current weights to the value dictated
   by the elapsed time. The LBP program must apply the correct weight
   at transaction time regardless of how recently the last poke
   occurred.
5. After the sale end timestamp passes, or when the sale creator
   explicitly closes the sale, the creator can withdraw:
   - The collateral raised (net of fees).
   - Any unsold project tokens remaining in the pool.
6. The sale creator can pause buying at any time during the sale
   period (emergency stop). Pausing does not affect weight
   progression; the weight schedule continues during a pause.
7. The sale creator can enable an optional allowlist gate for the
   sale. When enabled, only participants who can prove inclusion in
   the committed eligibility set may buy from the pool. Two gating
   modes are supported:
   - **Privacy-preserving gate** (preferred): eligibility is
     committed as a ZK-friendly accumulator; participants prove
     membership without revealing their address, using the mechanism
     delivered by
     [LP-0003](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0003.md).
     If LP-0003 is not yet delivered, this mode is optional for
     this submission.
   - **Merkle-proof gate** (required fallback): eligibility is
     committed as a Merkle root; participants submit an inclusion
     proof at buy time. This is an acceptable baseline if LP-0003
     is not yet available; the submission must document the privacy
     limitations of this approach compared to the ZK gate.
8. Slippage protection: buyers can specify a minimum token output
   amount. The transaction reverts if the execution price would
   produce fewer tokens than the specified minimum.
9. Use Associated Token Accounts (ATAs) for all token interactions,
   consistent with
   [LP-0014](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
   and [RFP-008](./RFP-008-lending-borrowing-protocol.md).

#### Usability

1. Provide an SDK for building Logos modules that interact with the
   launchpad program. The SDK must expose the full lifecycle for both
   participants (discover active sales, buy, query position) and
   creators (create sale, pause/resume, close, withdraw). The SDK
   must support both direct public account interaction and the
   deshield→buy→re-shield pattern for private account interaction.
   When the private account path is used, the SDK must handle the
   atomic deshield — both collateral and gas — as a single
   indivisible user action.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo. The mini-app must cover:
   - **Participant view**: browse active sales with live price,
     current token/collateral weight, time remaining, and total
     raised; execute a buy; view purchase history.
   - **Creator view**: create a new sale (all parameters including
     allowlist configuration), monitor an active sale,
     pause/resume, close sale, and withdraw proceeds.
3. The mini-app must display a pre-buy confirmation summary before
   each purchase: current token price, estimated tokens to be
   received, price impact, collateral spent, and fee.
4. When using the private account path, the mini-app must display a
   privacy disclosure before each buy, identifying what will be
   visible on-chain (buy transaction, collateral amount, tokens
   received, pool address, ephemeral account) and what will not be
   traceable (the buyer's private account and the subsequent
   destination of purchased tokens).
5. When using the private account path, the mini-app must enforce
   the atomic deshield — preventing a buyer from funding the
   ephemeral account from any external source, which would create
   an on-chain link to an existing identity.
6. When using the private account path, the mini-app must confirm
   before each buy that the buyer's shielded balance covers both
   the collateral amount and the gas fee within the single deshield
   action. A clear, actionable error must be shown if the balance
   is insufficient, preventing a partial deshield that could leave
   funds stranded.
7. Provide a sale analytics view showing, for each active or
   completed sale: total collateral raised, token price over time
   (price chart), number of buy transactions, and current pool
   composition. Analytics must not expose individual participant
   identities or link buy transactions to specific accounts.
8. Provide an IDL for the launchpad program using the
   [SPEL framework](https://github.com/logos-co/spel).
9. Failed or rejected buys must return clear, actionable error
   messages (e.g., insufficient balance, sale not yet started, sale
   ended, allowlist gate rejected, slippage exceeded).

#### Reliability

1. Pool state (weights, token balances, collateral raised) must
   remain consistent under concurrent buy submissions. No
   double-spend or incorrect pool accounting.
2. A failed buy must revert atomically — the buyer's collateral is
   not consumed and the pool state is unchanged.
3. Weight updates (pokes) must be idempotent: submitting multiple
   pokes within the same block or timestamp window must not corrupt
   pool state.
4. The minimum raise check at close time must be evaluated against
   final pool state, not an intermediate snapshot. If the minimum
   is not met, all collateral is returned to the creator and all
   project tokens are returned to the creator (not burned).

#### Performance

1. A single buy transaction completes within one LEZ transaction.
2. A weight poke completes within one LEZ transaction.
3. Document the compute unit (CU) cost of each operation: create
   sale, buy, poke weights, pause/resume, close sale, withdraw.
   Note the LEZ testnet version against which measurements were
   taken.

#### Supportability

1. The launchpad program is deployed and tested on LEZ
   devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer
   (standalone mode) and are included in CI. CI must be green on
   the default branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test. Test
   coverage must include: happy-path buy, slippage revert, allowlist
   gate accept and reject, sale close before end time, minimum raise
   not met refund path, weight poke at multiple points in the
   schedule.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for both creators and
   participants via CLI and mini-app.

#### + Privacy

1. The mini-app and SDK must support both direct public account
   interaction and the deshield→buy→re-shield pattern for private
   account interaction. When a user chooses the private account
   path, the SDK must enforce the complete deshield→buy→re-shield
   pattern — the re-shield step must not be skippable.
2. When using the private account path, the mini-app must display
   the pre-confirmation privacy summary described in the Usability
   section before each buy operation, identifying what is visible
   on-chain (buy amount, pool address, ephemeral intermediary
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

All LBP pool state is public on-chain (token pair, weights, price,
cumulative volume, total collateral raised). This is a deliberate
architectural choice: public state enables permissionless price
discovery, composability, and sale analytics without cryptographic
complexity in the pool itself.

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
2. Account A executes the buy against the LBP pool.
3. Account A re-shields the purchased project tokens to the buyer's
   private account. Account A is never reused.

> **Gas:** Both collateral and gas must come exclusively from the
> deshield in step 1. Funding account A from any external source
> — such as a CEX withdrawal or a known wallet — creates an
> on-chain link to an existing identity and breaks the privacy
> guarantee. The SDK must make this impossible; the atomic deshield
> is a single, indivisible user action.

#### What is public (observable on-chain)

- All pool state: token pair, current weights, price, total
  collateral raised, total tokens sold, sale start/end timestamps.
- All buy transactions: collateral spent, tokens received, and
  timestamp. When using the private account path, the buyer's
  address is an ephemeral intermediary account with no prior
  on-chain history.
- Allowlist gate configuration (Merkle root or ZK accumulator
  commitment) and whether the gate is enabled — but not the list
  of eligible addresses.
- Sale close and creator withdrawal transactions.

#### What is private (when using the private account path)

- Which private account originated the collateral for a buy.
- Where purchased tokens go after re-shielding.
- Any link between multiple buys by the same buyer (no on-chain
  linkability across ephemeral accounts).
- Whether a specific private account participated in the sale
  at all.

### Known Limitations

**Per-wallet buy limits are not supported.** Enforcing a cap on how
much collateral a single participant can contribute requires the
program to track cumulative spend per address. This is straightforward
for public account interactions, but is fundamentally incompatible
with the private account path: each buy from a private account uses
a fresh ephemeral public address with no prior history, so the program
cannot link multiple buys to the same underlying participant. Providing
this feature would either require breaking the privacy guarantee or
introducing off-chain coordination that reintroduces centralisation.
Projects that need whale concentration limits should use the allowlist
gate to restrict the eligible set instead.

### Soft Requirements

- Support for a fixed-price sale mode (no weight shift; constant
  price over duration), as an alternative to the LBP mechanism,
  for projects that prefer a known token price.
- Integration with the token vesting program (see
  [RFP-007](./RFP-007-token-vesting.md)): at sale close, purchased
  tokens can be routed directly into a vesting schedule rather than
  returned to the buyer's private account. This enables
  sale-plus-vesting in a single flow.
- Support for a Dutch auction mode (price declines on a configurable
  curve; first buyer at a given price wins that tranche).

## ⚠ Platform Dependencies

This RFP remains in **draft** until the dependencies below are
resolved. LEZ has similar programming capabilities to Solana but
several primitives required by a token launchpad are not yet
available.

### Hard blockers

These must be available on LEZ before this RFP can open.

#### On-chain clock / timestamp

LEZ does not yet have on-chain block time. The LBP mechanism is
fundamentally time-driven: pool weights shift linearly from the
start weight to the end weight over the sale duration, and the
current price at any moment is a function of elapsed time since
sale start. Without a reliable on-chain timestamp, weight
progression cannot be computed, and sale start and end timestamps
cannot be enforced. The launchpad cannot function without this
primitive.

#### General cross-program calls (LP-0015)

LEZ uses a tail-call execution model with no return. A buy
operation must: (1) call the token program to transfer collateral
from the buyer into the pool, (2) compute the token output based
on current weights, (3) call the token program again to transfer
project tokens to the buyer, and (4) update pool state (weights,
balances, cumulative volume). Without general cross-program calls,
the execution cannot continue after the first token program call,
making it impossible to complete the full buy atomically.

[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this. This
prize is currently **open**.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission (LP-0012)

Analytics dashboards and sale monitoring services need to react to
pool events (buy executed, weights updated, sale closed). Without
structured events, off-chain services must poll all accounts, which
is expensive and unreliable.

[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently
**open**.

## 👤 Recommended Team Profile

Team experienced with:

- AMM and weighted pool design (Balancer-style or equivalent)
- Solana or SVM program development (Anchor or native)
- DeFi smart contract security and auditing practices
- Front-end development for token sale and trading applications
- ZK membership proofs (if targeting the privacy-preserving
  allowlist mode — see LP-0003)

## ⏱ Timeline Expectations

Estimated duration: **14–16 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
  (reference for the optional public/private account interaction
  pattern)
- [RFP-007 — Privacy-Preserving Token Vesting](./RFP-007-token-vesting.md)
  (soft requirement: post-sale vesting integration)
- [LP-0003 — Private Allowlist / Airdrop Distributor](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0003.md)
  (privacy-preserving allowlist gate dependency)
- [LP-0013 — Token program improvements: mint authorities](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
- [LP-0014 — Token program improvements: ATAs + wallet tooling](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
- [Balancer LBP documentation](https://balancer.gitbook.io/balancer/smart-contracts/smart-pools/liquidity-bootstrapping-faq)
- [Fjord Foundry LBP reference](https://help.fjordfoundry.com/fjord-foundry-docs/for-sale-creators/faqs-creators/lbp-faq)
- [SPEL framework](https://github.com/logos-co/spel)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
