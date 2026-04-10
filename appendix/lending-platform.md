---
title: Lending Protocol Platform Context
status: draft
---


# Lending Protocol Platform Context

This appendix covers shared architectural context for the lending
protocol RFPs ([RFP-008](./RFP-008-lending-borrowing-protocol.md) and
[RFP-012](./RFP-012-advanced-lending-features.md)).


## Privacy Architecture

The protocol state (lending markets, positions, interest indices, and
token accounts) lives in **public (on-chain) state**. This is a
deliberate architectural choice: public state enables permissionless
liquidation, oracle integration, and composability without open
cryptographic research challenges.

User privacy is optionally enforced at the UX layer. The mini-app and
SDK support both direct public account interaction and private account
interaction via the deshield→interact→reshield pattern. When users opt
to interact from a private account, the SDK must enforce the complete
pattern as described below.

### Interaction flow

For every protocol operation (supply, borrow, repay, withdraw):

1. The user initiates the action from their private account. The SDK
   deshields to a **fresh, single-use** public account (account A)
   with no prior on-chain history. The deshield atomically transfers
   both the operation token **and** enough native token for gas in a
   single indivisible action.
2. Account A executes the protocol operation (e.g. supplies assets to
   a market).
3. Account A shields any outputs (withdrawn assets, repaid collateral)
   back to the user's private account. Account A is never reused.

> **Gas:** Both the operation token and gas must come exclusively from
> the deshield in step 1. Funding account A from any external source
> (such as a CEX withdrawal or a known wallet) creates an on-chain
> link to an existing identity and breaks the privacy guarantee. The
> SDK must make this impossible; the atomic deshield is a single,
> indivisible user action.

### What is public (observable on-chain)

- All market state: asset pairs, interest rate parameters, total
  supplied, total borrowed, utilisation, supply APY, borrow APR.
- All positions: collateral amounts, debt amounts, position LTV,
  attributed to the ephemeral intermediary account, not the
  private user.
- All protocol operations and their amounts.
- Oracle price feeds and liquidation events.

### What is private

- Which private account funded any supply or borrow.
- Where withdrawn assets or other outputs go after re-shielding.
- Any link between multiple protocol interactions by the same user
  (no on-chain linkability across ephemeral accounts).


## Platform Dependencies

These RFPs are open for proposal submission. However, development is
blocked until the dependencies below are resolved. LEZ has similar programming capabilities to Solana but
several primitives required by a lending protocol are not yet
available.

### Hard blockers

These must be available on LEZ before any lending RFP can open.

#### Oracle provider

No oracle provider is available on LEZ. The lending protocol requires
external price feeds for collateral valuation and liquidation triggers.

#### On-chain clock / timestamp

LEZ does not yet have on-chain block time. Interest accrual (the
core economic mechanic of a lending protocol) requires knowing how
much time has elapsed between interactions. Without a reliable on-chain
timestamp, interest rates, supply APY, and borrow APR cannot be
computed.

#### General cross-program calls (LP-0015)

LEZ uses a tail-call execution model rather than Solana's CPI
(Cross-Program Invocation). In Solana's model, a program can call
another program mid-execution and resume when the call returns. In
LEZ's model, a tail call hands off control entirely; there is no
return.

A lending operation like "supply" needs to: (1) call the token
program to transfer assets into the market, then (2) continue
executing to update interest indices and write position state. Without
general cross-program calls, step 2 cannot happen after step 1. Each
continuation would need to be a separate externally callable
entrypoint, which is fragile and insecure (anyone could call the
continuation directly, bypassing the token transfer).

[LP-0015](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0015.md)
(General cross-program calls via tail calls) solves this by introducing
internal-only entrypoints protected by an unforgeable capability, so
the lending program can tail-call the token program and have control
return to a protected continuation. This prize is currently **open**.

### Soft blockers

Desirable but the RFPs can open without them.

#### Event emission (LP-0012)

Liquidator bots and indexers need to monitor positions and react to
on-chain state changes. Without structured events, off-chain services
must poll all accounts, which is expensive and unreliable.

[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
(Structured events for LEZ program execution) is currently **open**.

### Risks

#### Compute budget

LEZ currently processes one private transaction per block. Liquidation
is the most compute-intensive lending operation: it reads the
borrower's position, collateral balance, oracle price, interest
indices, and market parameters, then writes updated state. On Solana,
a liquidation can consume 200,000 to 400,000 compute units. If LEZ's
per-transaction compute budget is insufficient, liquidations will fail
and the protocol risks insolvency. This needs benchmarking once the
protocol is under development.
