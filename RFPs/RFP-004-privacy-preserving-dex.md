---
id: RFP-004
title: Privacy-Preserving Decentralized Exchange (DEX)
tier: XL
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-004 — Privacy-Preserving Decentralized Exchange (DEX)

## 🧭 Overview

Build a decentralized exchange on LEZ that supports both public and
private accounts. Users trading from private accounts can swap tokens
without revealing their identities, order sizes, or trading strategies —
private account interactions are executed locally and submitted as
zero-knowledge proofs. The DEX must provide MEV-resistant, fair trading
for all participants.

A DEX is the most critical application for any new chain ecosystem.
On Ethereum, Uniswap has processed over $3.4 trillion in cumulative
volume and holds ~$6.8B in TVL; on Solana, Jupiter has routed over $1
trillion in cumulative volume, and Solana DEXes processed $326B in Q3
2025 alone. Trading is the primary activity that bootstraps economic
activity on any chain.

On transparent chains, this trading comes with severe downsides:
front-running and sandwich attacks extract hundreds of millions in MEV
from ordinary users. On LEZ, trades from private accounts are executed
locally and only a ZK proof of valid execution is pushed to the chain —
observers cannot see the trade details, eliminating these attack vectors
by design. This is not merely a privacy improvement; it is a
fundamentally fairer trading environment and a key differentiator for
the Logos ecosystem.

The team building this should have deep experience in AMM or order-book
design, SVM program development, and MEV-resistant trading mechanisms.

## 🔥 Why This Matters

Without a DEX on LEZ, users who bridge assets and hold them in private
accounts have no way to trade without moving funds off-chain or to a
centralised exchange, breaking the privacy guarantees that private
accounts provide. A LEZ-native DEX is the missing link between bridging
assets in and participating in a private economy.

Privacy also enables structural fairness. On Ethereum, ~$290M was
extracted via sandwich attacks in 2025; on Solana, $370–500M over a
16-month period. While solutions like Flashbots (Ethereum) and Jito's
DontFront (Solana) mitigate MEV, they are afterthoughts bolted onto
transparent systems. On LEZ, MEV resistance is a first-class property
of the execution environment, making the DEX inherently fairer for all
participants.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

1. Implement an automated market maker (AMM) program on LEZ that
   supports both public and private accounts.
2. Support creation of liquidity pools for arbitrary token pairs.
3. Liquidity providers using private accounts can add and remove
   liquidity without revealing their positions or addresses.
4. Traders using private accounts can swap tokens without revealing
   trade size, direction, or identity.
5. Traders and LPs using public accounts can interact with the same
   pools; their transactions are executed transparently on-chain
   (standard public account behaviour).
6. The pool creator selects a fee tier at pool creation time (e.g.,
   0.01%, 0.05%, 0.3%, 1%); the fee tier is immutable per pool.
   Multiple pools for the same token pair with different fee tiers
   can coexist. Trading fees are paid by the trader and distributed
   to LPs.
7. Implement slippage protection with user-configurable tolerance and
   minimum output guarantees.
8. Include a freeze/emergency-pause mechanism for individual pools
   (see [RFP-002](./RFP-002-freeze-authority-poc.md)).
9. Use Associated Token Accounts (ATAs) for all token interactions —
   pool token accounts, LP token accounts, and trader token accounts
   must use the deterministic ATA derivation per `(owner, mint)` pair
   (see [LP-0014](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0014.md)).

#### Usability

1. Provide an SDK that can be used to build Logos modules for
   interacting with the DEX (swapping, pool creation, liquidity
   management).
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a pool analytics view showing aggregate volume, TVL, and
   fee revenue without revealing individual positions.
4. The swap interface and documentation must clearly communicate to the user what
   information is public vs. private for each action (e.g., "pool
   selection is public, trade size is private").
5. Failed or rejected swaps must return clear, actionable error messages.
6. Provide an IDL for the DEX program, preferably using the
   [SPEL framework](https://github.com/logos-co/spel).

#### Reliability

1. Pool state must remain consistent under concurrent swap submissions;
   no double-spend or incorrect pool balance.
2. If the DEX program is frozen via the emergency-pause mechanism,
   liquidity withdrawal remains functional so that LPs can exit.

#### Performance

1. A swap against an existing pool completes within a single LEZ
   transaction.
2. The compute unit usage of each operation (swap, add/remove
   liquidity, pool creation) must be documented; LEZ's per-transaction
   compute budget may change during testnet.
3. Pool creation and liquidity operations complete within a single
   transaction each.

#### Supportability

1. The DEX program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone
   mode) and are included in CI — CI must be green on the default
   branch.
3. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
4. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for interacting with the
   DEX via CLI and front-end (pool creation, swapping, LP management,
   emergency pause).

### Soft Requirements

1. **Concentrated liquidity**: Support for concentrated liquidity
   positions (Uniswap V3 / Orca Whirlpool style) where LPs select
   specific price ranges, improving capital efficiency.
2. **Limit orders**: Support for limit orders, potentially executed
   within a batch auction framework.
3. **Multi-hop routing**: Router functionality for multi-hop swaps
   across pools when a direct pair does not exist.
4. **DCA (Dollar-Cost Averaging)**: Scheduled recurring swaps at
   user-defined intervals, executing privately.
5. **Protocol fee**: An admin-configurable protocol fee that redirects
   a fraction of trading fees to a designated account (e.g., for
   ecosystem funding or governance treasury).
6. **Atomic swap integration**: Support for cross-chain atomic swaps,
   enabling trustless trading with assets on other chains without a
   bridge (builds on LEZ atomic swap primitives being developed
   separately).

### Public vs. Private Interaction Model

LEZ cleanly separates public and private state while keeping them
interoperable. The DEX must support both account types against the same
liquidity pools.

**Public accounts** have their state on-chain. When a public account
swaps or provides liquidity, the transaction is executed by validators
transparently — the account ID, amounts, and direction are visible
on-chain, as on any transparent chain.

**Private accounts** store their state off-chain. When a private account
swaps or provides liquidity, the wallet executes the AMM logic locally,
generates a ZK proof of correct execution, and submits the proof to
validators. Only commitments (hash-like bindings to the new state) and
nullifiers (marking the previous state as spent) appear on-chain — the
account identity, trade size, direction, and balances remain local to
the account owner.

**What is always public (pool-level state):**
- Existence of liquidity pools (token pair, fee tier).
- Aggregate pool state: total TVL, cumulative volume, current price
  (derived from pool reserves ratio).
- That a transaction occurred (the proof/transaction exists on-chain).

**What is private (for private account interactions):**
- Account identity (which private account initiated the action).
- Trade size and direction (buy vs. sell, amount in/out).
- Individual LP position size and price range.
- Individual fee earnings and withdrawal amounts.
- Order flow and trading strategy patterns.

**Design Constraint:** Because private account activity is not visible
on LEZ, per-account features like freezing a specific trader's access
are not feasible (see RFP-002 out-of-scope note). Any freeze mechanism
must operate at the pool or program level, not the account level.

## 👤 Recommended Team Profile

Team experienced with:

- AMM and/or order-book DEX design and implementation
- Solana or SVM program development (Anchor or native)
- MEV research and mitigation techniques
- DeFi smart contract development and security auditing
- Front-end development for trading applications
- Zero-knowledge proof systems (ZK circuits, commitment schemes)

## ⏱ Timeline Expectations

Estimated duration: **14–22 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
