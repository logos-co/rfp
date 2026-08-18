---
id: RFP-004
title: Privacy-Preserving Decentralized Exchange (DEX)
tier: XL
funding: $XXXXX
status: closed
dependencies:
  - id: LP-0013
    reason: Token transfer-authority primitives are required for the DEX program to custody pool reserves, pay swap output, return LP deposits, and route trading fees to LPs and the protocol treasury.
  - id: RFP-001
    reason: Provides the standardised admin authority library that governs each AMM namespace (trading fee, protocol fee, and treasury address per F.6, F.11, F.13) and the TWAP accumulator surface integrated per F.10 (per-pool tick-delta clamp, observation cardinality registration, as defined by RFP-019).
  - id: LP-0015
    reason: General cross-program calls via tail calls, used to compose token transfers with reserve and state updates within a single atomic swap.
  - id: LP-0014
    reason: Associated Token Accounts for user-facing token accounts (requirement F.8).
  - id: LP-0012
    reason: Structured event emission, used by the pool analytics view and third-party indexers to observe pool state changes.
category: Applications & Integrations
---

# RFP-004 — Privacy-Preserving Decentralized Exchange (DEX)

## 🧭 Overview

Build a decentralized exchange on LEZ with public AMM liquidity pools. Users
with public accounts interact with the DEX directly. Users with private accounts
interact via the deshield→swap→re-shield pattern: the SDK deshields tokens to a
fresh ephemeral public account, executes the swap in a public pool, and
re-shields the output back to the user's private account. When interacting from
a private account, the origin and destination of funds are not traceable
on-chain, protecting user identity without requiring private pool state.

A DEX is the most critical application for any new chain ecosystem. On Ethereum,
Uniswap has processed over $3.4 trillion in cumulative volume and holds ~$6.8B
in TVL; on Solana, Jupiter has routed over $1 trillion in cumulative volume, and
Solana DEXes processed $326B in Q3 2025 alone. Trading is the primary activity
that bootstraps economic activity on any chain.

On transparent chains, this trading comes with severe downsides: front-running
and sandwich attacks extract hundreds of millions in MEV from ordinary users. On
LEZ, when users interact from a private account, their identity is never linked
to a swap on-chain — observers see a trade from an ephemeral public account with
no prior history, making identity-based front-running and wallet-profiling
impossible. Sandwich attacks, back-running, and CEX-DEX arbitrage remain
possible as they depend on trade size and pool state rather than user identity;
mitigating these is out of scope for this RFP. This is a meaningful privacy
improvement and a key differentiator for the Logos ecosystem.

The team building this should have deep experience in AMM or order-book design,
SVM program development, and MEV-resistant trading mechanisms.

## 🔥 Why This Matters

Without a DEX on LEZ, users who bridge assets and hold them in private accounts
have no way to trade without moving funds off-chain or to a centralised
exchange, breaking the privacy guarantees that private accounts provide. A
LEZ-native DEX is the missing link between bridging assets in and participating
in a private economy.

Privacy also enables structural fairness. On Ethereum, ~$290M was extracted via
sandwich attacks in 2025; on Solana, $370–500M over a 16-month period. While
solutions like Flashbots (Ethereum) and Jito's DontFront (Solana) mitigate MEV,
they are afterthoughts bolted onto transparent systems. On LEZ, MEV resistance
is a first-class property of the execution environment, making the DEX
inherently fairer for all participants.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

01. Implement an automated market maker (AMM) program on LEZ with public
    liquidity pools supporting the deshield→swap→re-shield interaction pattern
    for privacy-preserving trading.
02. Support creation of liquidity pools for arbitrary token pairs.
03. Liquidity providers can add and withdraw liquidity directly from a public
    account, or via the deshield→interact→reshield pattern from a private
    account. The LP position is public on-chain; when using a private account,
    which private account originated or received the funds is not traceable.
04. Traders can swap tokens directly from a public account, or via the
    deshield→swap→re-shield pattern from a private account. Trade size and
    direction are visible on-chain; when using a private account, which private
    account originated the funds or where they go after re-shielding is not
    traceable.
05. Traders and LPs using public accounts can interact with the same pools;
    their transactions are executed transparently on-chain (standard public
    account behaviour).
06. Trading fees are paid by the trader on every swap. The fee rate is set by
    the namespace admin authority and applies uniformly to all pools in that
    namespace; it is not selected per pool and is updatable after deployment.
    The fee is split between LPs and the protocol treasury per F.11.
07. Implement slippage protection with user-configurable tolerance and minimum
    output guarantees.
08. The DEX program must be compatible with Associated Token Accounts (ATAs) for
    user-facing token accounts: when a trader or LP supplies an ATA derived per
    `(owner, mint)` pair (see
    [LP-0014](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0014.md)),
    the program must accept it without requiring an alternative derivation. ATAs
    must not be forced on users; the program must also accept any valid SPL
    token account owned by the caller. Pool-side vault accounts may use
    program-derived addresses (PDAs) rather than ATAs, matching Solana DEX
    practice.
09. Implement a permissionless `sync()` function that updates a pool's cached
    reserves to match the actual vault token balances, absorbing any surplus
    from unsolicited transfers into the pool for the benefit of LPs. See
    [Appendix: DEX Ecosystem Behaviour, section 10](../appendix/dex-ecosystem-behaviour.md#10-reserve-reconciliation-sync-and-skim)
    for rationale and ecosystem precedent.
10. Integrate the TWAP accumulator component delivered by
    [RFP-019](./RFP-019-twap-oracle.md): every state-changing pool operation
    (swap, add liquidity, remove liquidity) invokes the component so the pool's
    price accumulators are maintained. If the component is not yet available
    when the DEX is delivered, the program must expose the integration point so
    the component can be hooked in later without redesigning pool state.
11. The trading fee of F.6 is split into an LP share and a protocol fee. The
    protocol fee is a subpart of the trading fee, not an additional charge on
    the trader. The namespace admin authority sets the protocol fee as a
    fraction of the trading fee, along with the treasury address that receives
    it, and both are updatable. The program does not restrict the trading fee or
    the protocol fee to a fixed range or a set of preset tiers; the only bound
    is that the protocol fee cannot exceed the trading fee. Either may be zero.
12. Accrued protocol fees are tracked separately from pool reserves rather than
    inferred from the difference between vault balance and reserves, so that fee
    accounting and LP accounting cannot corrupt each other. Withdrawing accrued
    protocol fees must not draw on LP principal.
13. A single deployment of the program supports any number of independent AMM
    namespaces. Anyone can permissionlessly create a namespace, seeding it with
    its own admin authority, trading fee, protocol fee, treasury, and pools.
    Creating a namespace does not require redeploying or modifying the program,
    nor permission from the deployer or from any existing namespace.
14. Namespaces share no global or singleton state. Pool addresses are derived
    such that pools of different namespaces never collide for the same token
    pair. Every state-changing instruction resolves pool, vault, treasury, and
    admin accounts against the namespace the pool belongs to, and rejects
    accounts belonging to another namespace. An admin authority has no power
    over any namespace other than its own.

#### Usability

01. Provide an SDK that can be used to build Logos modules for interacting with
    the DEX (swapping, pool creation, liquidity management). When the user
    interacts from a private account, the SDK must handle the atomic deshield —
    transferring both the swap token and a small amount of native token for gas
    — as a single indivisible action, preventing accidental privacy leaks from
    externally funding account A.
02. Provide a Logos mini-app GUI with local build instructions, downloadable
    assets, and loadable in Logos app (Basecamp) via git repo.
03. Provide a CLI that covers core functionality of the program (pool creation,
    swapping, LP management). The CLI may have fewer features than the GUI
    mini-app but must support all essential operations.
04. Provide an IDL for the DEX program, preferably using the
    [SPEL framework](https://github.com/logos-co/spel).
05. Provide a pool analytics view showing aggregate volume, TVL, and fee revenue
    without revealing individual positions.
06. Documentation must clearly explain what information is public vs. private
    for each action (trade size and pool used are visible on-chain; the private
    account that originated or receives the funds is not traceable).
07. Failed or rejected swaps must return clear, actionable error messages.
08. Before each swap or liquidity operation, the mini-app must show the
    estimated transaction fee. When the user interacts from a private account,
    it must also confirm that the shielded balance covers both the operation
    amount and fees within the single deshield action; a clear, actionable error
    must be shown if the balance is insufficient (preventing partial deshields
    that could leave funds stranded in an ephemeral account).
09. The mini-app must display a swap preview before the user confirms: estimated
    output amount, effective price, price impact, and fee taken, so the user can
    evaluate the trade before confirming.
10. The SDK, CLI, and mini-app let the caller select which namespace to operate
    against, and the mini-app shows the active namespace. Pools of different
    namespaces are never mixed in quotes, routing, or LP position listings.
11. The mini-app and CLI show the current trading fee and protocol fee for the
    namespace in use. The swap preview of U.9 breaks the fee into the LP share
    and the protocol share. The pool analytics view of U.5 reports protocol fee
    revenue separately from LP fee revenue.
12. The SDK, CLI, and mini-app expose namespace creation and the admin
    operations for a namespace: setting the trading fee, the protocol fee, and
    the treasury address, plus the admin authority transfer and renunciation
    operations of RFP-001. An admin operation attempted without the admin
    authority fails with a clear, actionable error.

#### Reliability

1. Pool state must remain consistent under concurrent swap submissions; no
   double-spend or incorrect pool balance.
2. An operation on a pool of one namespace never reads or writes the state,
   vaults, or treasury of another namespace, including when supplied with
   deliberately mismatched accounts from a second namespace.
3. For every swap, the LP share and the protocol share sum to the trading fee
   charged to the trader, with rounding resolved in favour of the pool rather
   than the treasury.
4. A trading fee or protocol fee update applies only to fees accrued after the
   update, never retroactively to fees already earned by LPs. The split applied
   to a swap is the one in effect when the swap executes, and the swap respects
   the trader's slippage bound of F.7 regardless.

#### Performance

1. A swap against an existing pool completes within a single LEZ transaction.
2. Pool creation and liquidity operations complete within a single transaction
   each.
3. Compute unit usage and transaction size of each operation (swap, add
   liquidity, remove liquidity, pool creation) must be documented and
   benchmarked against LEZ devnet limits; LEZ's per-transaction compute budget
   and block size may change during testnet.

#### Supportability

1. The DEX program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   and are included in CI.
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.
5. A README documents end-to-end usage: deployment steps, program addresses, and
   step-by-step instructions for interacting with the DEX via CLI and front-end
   (pool creation, swapping, LP management). It must also document how to create
   a namespace, how namespace and pool addresses are derived, and how the admin
   authority configures the trading fee, protocol fee, and treasury.
6. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the SDK, covering the developer integration journey for pool creation,
   swapping, and liquidity management.
7. Submit a
   [doc packet](https://github.com/logos-co/logos-docs/issues/new?template=doc-packet.yml)
   for the CLI, covering the core operator/user journey.
8. Provide Figma designs or equivalent for the mini-app GUI.

#### + Privacy

1. The mini-app and SDK must support both direct public account interaction and
   the deshield→swap→re-shield pattern for private account interaction. When a
   user interacts from a private account, the SDK must enforce the complete
   deshield→swap→re-shield pattern — the re-shield step must not be skippable.
2. When interacting from a private account, the mini-app must display a
   pre-confirmation summary for each operation that clearly identifies what will
   be visible on-chain (trade size, direction, pool address, ephemeral
   intermediary account) and what will remain private (the originating private
   account, the destination of re-shielded tokens, and any link between separate
   swaps by the same user).
3. When interacting from a private account, the SDK must validate that the
   target account for re-shielding swap output is a private (shielded) account
   before submitting the transaction, and reject the operation with an explicit
   error if it is not.
4. The ephemeral public account (account A) created during the deshield step
   must never be reused across operations. Each swap or liquidity operation from
   a private account must use a freshly generated account with no prior on-chain
   history.

### Soft Requirements

If possible.

#### Functionality

1. Support multi-hop routing across multiple pools within a single transaction
   (e.g. flash-accounting style settlement of intermediate hops), reducing
   slippage on token pairs without a direct pool.

### Out of Scope

The following are explicitly excluded from this RFP:

- A `skim()` or `recoverSurplus()` instruction that extracts surplus tokens from
  a pool's vault to a caller-specified address. Surplus reconciliation is
  handled exclusively by the permissionless `sync()` function (Functionality
  requirement F.9), which folds surplus into the pool to benefit LPs. Among
  surveyed protocols, only Uniswap V2 exposes a `skim()`-style instruction;
  Uniswap V4, Balancer V3, Curve StableSwapNG, Raydium, and Orca Whirlpools do
  not. See
  [Appendix: DEX Ecosystem Behaviour, section 10](../appendix/dex-ecosystem-behaviour.md#10-reserve-reconciliation-sync-and-skim).

### Fee Structure and Namespaces

The trading fee is paid by the trader; the protocol fee is a subpart of it, not
an additional charge. A protocol fee of zero leaves the entire trading fee with
LPs. Both are set by the namespace admin authority and apply uniformly to every
pool in that namespace.

The program is deployed once. After that, anyone can create a namespace with
their own seed, admin authority, fees, treasury, and pools, without permission
from the deployer. This is what allows several independent AMMs to coexist on
the chain, and it keeps the reach of any admin authority limited to its own
namespace. Fee ranges are left open for the same reason: a namespace admin
setting an uncompetitive fee affects only their own pools, and LPs and traders
can move to another namespace or create one.

Accrued protocol fees are tracked in their own state rather than derived from
vault balance minus reserves. Curve StableSwapNG uses this separation and
documents its failure mode: fee state that is immune to a negative rebase can be
withdrawn ahead of LPs, leaving LPs to absorb the shortfall (MixBytes audit,
October 2023). Proposals should address withdrawal ordering.

### Privacy Architecture

All DEX liquidity pools are public on-chain state. User privacy is enforced at
the UX layer for private account users. The mini-app and SDK support both direct
public account interaction and private account interaction via the
deshield→swap→re-shield pattern. When a user interacts from a private account,
the SDK must enforce the complete pattern as described below.

#### Interaction flow

For every protocol operation initiated from a private account (swap, add/remove
liquidity):

1. The user initiates the action from their private account. The SDK deshields
   to a **fresh, single-use** public account (account A) with no prior on-chain
   history. The deshield atomically transfers both the operation token **and**
   enough native token for gas in a single indivisible action.
2. Account A executes the operation in a public pool.
3. Account A shields any outputs (swap proceeds, withdrawn liquidity) back to
   the user's private account. Account A is never reused.

> **Gas:** Both the operation token and gas must come exclusively from the
> deshield in step 1. Funding account A from any external source — such as a CEX
> withdrawal or a known wallet — creates an on-chain link to an existing
> identity and breaks the privacy guarantee. The SDK must make this impossible;
> the atomic deshield is a single, indivisible user action.

#### What is public (observable on-chain)

- All pool state: token pair, total TVL, cumulative volume, current price, and
  the namespace the pool belongs to.
- All namespace state: admin authority, trading fee, protocol fee, treasury
  address, and accrued protocol fee revenue.
- All swap and liquidity transactions: trade size, direction, and the
  originating account address (the ephemeral intermediary account for private
  account interactions, the user's public account otherwise).
- LP position sizes and fee earnings.

#### What is private

- Which private account originated the funds for a swap or LP deposit.
- Where output tokens go after re-shielding.
- Any link between multiple operations by the same user (no on-chain linkability
  across ephemeral accounts).

## ⚠ Platform Dependencies

This RFP is open for proposals. Proposers may begin design and development work,
but a working on-chain deployment depends on the primitives below. The privacy
primitives are core LEE features; the token-program and runtime primitives are
tracked as lambda prizes.

### Hard blockers

These must be available on LEZ before the DEX can hold liquidity and settle
swaps on-chain.

#### Token authorities (LP-0013)

The DEX program is a token custodian: it holds pool reserves for each token
pair, pays swap output to traders, returns deposits to LPs on withdrawal, and
routes trading fees to LPs and the protocol treasury. This requires the
transfer-authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md),
currently **open**.

#### Admin authority (RFP-001)

Each AMM namespace is governed by its own admin authority (Functionality
requirement F.13), which configures the trading fee (F.6), the protocol fee, and
the treasury address (F.11), and can be transferred or renounced. The TWAP
accumulator component integrated per F.10 carries further admin-governed
parameters defined by [RFP-019](./RFP-019-twap-oracle.md): the per-pool
tick-delta clamp (`MAX_TICK_DELTA`) and observation cardinality registration are
owner-gated. Managing all of these uses the standardised admin authority library
from [RFP-001](./RFP-001-admin-authority-lib.md). The RFP is closed (candidate
picked) and the library is in development.

### Resolved dependencies

These primitives were once blockers but are now delivered on LEZ, so they no
longer gate this RFP. They remain in the frontmatter `dependencies` index for
traceability.

#### General cross-program calls (LP-0015)

A swap must transfer the input token into the pool vault, compute the output
using the constant-product formula, transfer the output token to the trader, and
update cached reserves, all within one atomic transaction. General cross-program
calls via tail calls let the operation continue into a protected continuation
after each token transfer.
[LP-0015](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0015.md)
is **closed**, delivered by the LEZ team as part of the core runtime.

#### Associated Token Accounts (LP-0014)

User-facing token accounts use the deterministic ATA derivation per
`(owner, mint)` pair (Functionality requirement F.8).
[LP-0014](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0014.md)
is **closed**.

#### Event emission (LP-0012)

The pool analytics view (Usability requirement) and any third-party indexer
observe pool state changes (swaps, liquidity added or removed) through
structured events rather than polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0012.md)
is **closed**.

### Soft blockers

#### Oracle TWAP component (RFP-019)

The DEX is expected to hook the TWAP accumulator component delivered by
[RFP-019](./RFP-019-twap-oracle.md) into its pools so that every state-changing
pool operation maintains the price accumulators (Functionality requirement
F.10). The DEX can ship before the component is available by exposing the
integration point, so this is a soft dependency rather than a frontmatter entry.

### Privacy primitives

The deshield→swap→re-shield pattern relies on LEE private accounts and the
atomic deshield action described in the Scope of Work. These are core LEE
features rather than lambda prizes; proposers should confirm the current state
of private-account support on LEZ devnet against the Resources below before
relying on it.

### Risks

#### Compute budget

LEZ currently processes one private transaction per block (as of 2026-04). A
swap that deshields, transfers into the pool vault, computes output, transfers
output, re-shields, updates reserves, and accounts the protocol fee share is
compute-intensive. On Solana, where the per-instruction default is 200,000
compute units and the per-transaction maximum is 1.4M compute units (see
[Solana compute budget](https://solana.com/docs/core/fees/compute-budget)), a
multi-step settlement consumes hundreds of thousands of compute units; the exact
figure for LEZ depends on the implementation. If LEZ's per-transaction compute
budget is lower, multi-hop routing (the soft Functionality requirement) may not
fit in a single transaction. Performance requirement P.3 requires benchmarking
each operation against LEZ devnet limits.

## 👤 Recommended Team Profile

Team experienced with:

- AMM and/or order-book DEX design and implementation
- Solana or SVM program development (Anchor or native)
- MEV research and mitigation techniques
- DeFi smart contract development and security auditing
- Front-end development for trading applications
- Zero-knowledge proof systems (ZK circuits, commitment schemes)

## ⏱ Timeline Expectations

Estimated duration: **14 weeks** (Uniswap V2 equivalent with
deshield→swap→re-shield support).

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [LEZ programs](https://github.com/logos-blockchain/lez-programs): on-chain
  TWAP program reference implementation

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
