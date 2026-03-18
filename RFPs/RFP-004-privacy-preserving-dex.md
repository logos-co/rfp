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

Build a decentralized exchange on LEZ with public AMM liquidity pools.
Privacy is achieved through a deshield→swap→re-shield pattern: a user
deshields tokens from their private account to a fresh public account,
swaps in a public pool, and re-shields the output back to a private
account. The origin and destination of funds are not traceable
on-chain, protecting user identity without requiring private pool state.

A DEX is the most critical application for any new chain ecosystem.
On Ethereum, Uniswap has processed over $3.4 trillion in cumulative
volume and holds ~$6.8B in TVL; on Solana, Jupiter has routed over $1
trillion in cumulative volume, and Solana DEXes processed $326B in Q3
2025 alone. Trading is the primary activity that bootstraps economic
activity on any chain.

On transparent chains, this trading comes with severe downsides:
front-running and sandwich attacks extract hundreds of millions in MEV
from ordinary users. On LEZ, the deshield→swap→re-shield pattern means
a user's private account identity is never linked to a swap on-chain —
observers see a trade from an ephemeral public account with no prior
history, making identity-based front-running and wallet-profiling
impossible. This is a meaningful privacy improvement and a key
differentiator for the Logos ecosystem.

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

1. Implement an automated market maker (AMM) program on LEZ with
   public liquidity pools supporting the deshield→swap→re-shield
   interaction pattern for privacy-preserving trading.
2. Support creation of liquidity pools for arbitrary token pairs.
3. Liquidity providers can add liquidity from a fresh public account
   (funded by deshielding from a private account) and later withdraw
   to a private account via re-shielding. The LP position is public
   on-chain; what is not traceable is which private account originated
   or received the funds.
4. Traders can swap tokens via the deshield→swap→re-shield pattern.
   Trade size and direction are visible on-chain; what is not
   traceable is which private account originated the funds or where
   they go after re-shielding.
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
   (see [RFP-002](./RFP-002-freeze-authority-poc.md)). The freeze
   authority should be held by the pool creator or program deployer;
   the exact authority model is to be finalised once RFP-002 scope
   is confirmed.
9. Use Associated Token Accounts (ATAs) for all token interactions —
   pool token accounts, LP token accounts, and trader token accounts
   must use the deterministic ATA derivation per `(owner, mint)` pair
   (see [LP-0014](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0014.md)).

#### Usability

1. Provide an SDK that can be used to build Logos modules for
   interacting with the DEX (swapping, pool creation, liquidity
   management). The SDK must handle the atomic deshield — transferring
   both the swap token and a small amount of native token for gas — as
   a single user action, preventing accidental privacy leaks from
   externally funding account A.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a pool analytics view showing aggregate volume, TVL, and
   fee revenue without revealing individual positions.
4. The swap interface and documentation must clearly communicate to the
   user what information is public vs. private for each action (e.g.,
   "trade size and pool used are visible on-chain; the private account
   that originated or receives the funds is not traceable").
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
2. The transaction size of each operation (swap, add/remove
   liquidity, pool creation) must be documented; LEZ's block size is
   limited and this budget may change during testnet.
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

### Public vs. Private Interaction Model

All DEX liquidity pools are public on-chain state. Privacy is achieved
at the user interaction level via the deshield→swap→re-shield pattern,
leveraging LEZ's private account primitives.

**Privacy flow for a private swap:**
1. User deshields from a private account to a **fresh, single-use**
   public account (account A) with no prior on-chain history. The
   deshield atomically transfers both the swap token **and** a small
   amount of native token (to cover transaction fees) so that account
   A never needs to be funded from any external source.
2. Account A performs the swap in a public pool.
3. After the swap, account A performs a shielded transfer back into a
   private account. Account A is never reused after this point.

The result: the private account's identity is not exposed on-chain,
and the link between the origin of funds and the destination after
re-shielding is not traceable.

> **Gas Consideration:** Both the swap token and the native token for
> transaction fees must come exclusively from the deshield in step 1.
> Funding account A from any external source — such as a CEX withdrawal
> or a known wallet — would create an on-chain link between account A
> and an existing identity, breaking the privacy guarantee. The SDK
> must abstract this so that users cannot accidentally fund account A
> externally; the atomic deshield (tokens + native token for gas) must
> be a single, indivisible user action.

**What is always public (observable on-chain):**
- Existence of liquidity pools (token pair, fee tier).
- Aggregate pool state: total TVL, cumulative volume, current price
  (derived from pool reserves ratio).
- All swap and liquidity transactions: trade size, direction, and the
  public account address used (the ephemeral deshield intermediary).
- LP position sizes and fee earnings.

**What is private (when using the deshield→swap→re-shield pattern):**
- Which private account originated the funds for a swap or LP deposit.
- Where the output tokens go after re-shielding.
- The connection between multiple swaps performed by the same private
  account (no on-chain linkability).
- The connection between account A and any prior on-chain identity
  (account A is single-use and never funded from any external source).

**Design Constraint:** Because only public accounts interact directly
with pools, per-account features like freezing a specific trader's
access are not feasible. Any freeze mechanism must operate at the pool
or program level, not the account level.

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

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
