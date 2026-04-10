---
id: RFP-014
title: Liquidation & Auction Engine
tier: L
funding: $XXXXX
status: draft
dependencies: RFP-001 (Admin Authority), RFP-002 (Freeze Authority), RFP-013 (Reflexive Stablecoin Protocol or equivalent CDP host)
category: Applications & Integrations
---

# RFP-014 — Liquidation & Auction Engine

## Overview

Build a permissionless liquidation and auction system for collateralized debt position (CDP) protocols on the Logos Execution Zone (LEZ). This system is designed as a reusable program that each CDP product can deploy as its own instance. It detects undercollateralized positions, seizes collateral, and sells it via auctions to cover debt. It also manages protocol surplus and debt through separate auction mechanisms.

The design follows proven patterns from [RAI / Reflexer Finance](https://reflexer.finance/): a fixed-discount auction for seized collateral that incentivizes quick liquidation while minimizing protocol losses, plus surplus and debt auction houses for system-level balance management. For a survey of liquidation mechanisms and the design rationale behind this approach, see the [companion appendix](../appendix/appendix-liquidation-auction-ecosystem.md). When a position becomes undercollateralized, the system seizes its collateral and begins an auction where the discount to market price increases over time until the target amount is raised. Surplus from liquidation penalties and stability fees is auctioned for the reference token and burned. Shortfalls from insufficient collateral auctions are covered by minting the reference token and auctioning for stablecoins.

## Why This Matters

CDP protocols require liquidation to remain solvent. Without it, bad debt accumulates, threatening protocol solvency and user confidence. For the Logos ecosystem, a well-designed liquidation system protects solvency by closing positions before they become underwater, minimizes losses via efficient price discovery, and handles system imbalances via surplus and debt auctions.

Building this as a reusable program has ecosystem-wide benefits. Each CDP product can deploy its own instance of the same battle-tested liquidation code rather than rebuilding from scratch. This includes the reflexive stablecoin ([RFP-013](./RFP-013-reflexive-stablecoin-protocol.md)), future lending protocols, or Maker-style multi-collateral systems. Reusing the same codebase means shared security audits, shared liquidator bot logic, and faster time-to-market for new CDP products. Each instance is configured for its specific product while inheriting the proven auction mechanisms and safety guarantees.

## Scope of Work

### Hard Requirements

#### Functionality

1. Anyone can trigger liquidation on any undercollateralized position. No whitelist or special permissions required. The liquidator receives an incentive (e.g., a percentage of seized collateral).

2. Seized collateral is sold via auctions where the discount to market price increases over time. The discount schedule must be configurable. Auctions terminate early when the target amount is reached.

3. Auctions support buying partial collateral lots, preventing large positions from becoming stuck.

4. When collateral auctions don't raise enough to cover debt plus penalty, the shortfall is handled via a debt auction mechanism. The mechanism must be fully implemented and tested (using a mock token), but parameterized by a configurable token address so the deployer can point it at whatever protocol token they choose. The protocol token itself is out of scope (see [RFP-013 Out of Scope](./RFP-013-reflexive-stablecoin-protocol.md#out-of-scope)). Until a protocol token is configured, the surplus buffer absorbs shortfalls up to its capacity; if bad debt exceeds the buffer, the system pauses new debt generation via freeze authority ([RFP-002](./RFP-002-freeze-authority-poc.md)).

5. When the protocol accumulates excess stablecoin above a threshold, surplus auctions sell it for a protocol token (which is burned). As with F4, the surplus auction mechanism must be fully implemented and tested with a mock token, parameterized by a configurable token address. Until a protocol token is configured, surplus accumulates in the buffer without being auctioned.

6. Auctions follow strict state transitions with no reversals. All state changes are atomic and verifiable.

7. The system integrates with its host CDP protocol via a well-defined interface. Each product deploys its own configured instance of the program.

#### Usability

1. Build the program using the SPEL framework, which generates the IDL and client code from the program definition.

2. Provide a Logos mini-app GUI with local build instructions, downloadable assets, and loadable in Logos app (Basecamp) via git repo.

3. Provide a CLI that covers core functionality of the program. The CLI may have fewer features than the GUI mini-app but must support all essential operations.

4. Tools display undercollateralized positions with estimated liquidator incentives and allow users to trigger liquidation, enabling efficient bot and manual operation.

5. Active auctions display collateral type, amount, current discount, raised amount vs target, and time remaining. Users can bid with clear previews of received collateral and costs.

6. Provide an SDK that can be used to build Logos modules for interacting with the program. The SDK must support private account operations via the deshield-interact-reshield pattern.

#### Reliability

1. Auction operations are non-reentrant. State transitions are ordered to prevent manipulation.

2. Liquidation uses time-weighted or confidence-adjusted prices, not raw spot. If price feeds are stale, liquidations pause for that collateral type.

3. Multiple liquidations and auctions can occur simultaneously without interference. Each auction maintains isolated state.

4. If a price feed fails for one collateral type, only that type's liquidations pause.

5. Other collateral types and non-liquidation operations continue unaffected.

6. Critical invariants: seized collateral equals collateral in auction; debt shortfalls are properly accounted; surplus and debt auctions maintain system balance.

#### Performance

1. Liquidation transactions complete within one block.

2. The system supports many concurrent auctions without exceeding compute limits.

3. Bid transactions complete within one block.

#### Supportability

1. The program is deployed and tested on LEZ devnet/testnet.

2. End-to-end integration tests run against a LEZ sequencer (standalone mode) with `RISC0_DEV_MODE=0` and are included in CI.

3. CI must be green on the default branch.

4. Every hard requirement in Functionality, Usability, Reliability, and Performance has at least one corresponding test.

5. A README documents end-to-end usage: deployment steps, program addresses, and step-by-step instructions for interacting with the program via CLI and mini-app.

#### + Privacy

1. The mini-app and SDK must support both direct public account interaction and the deshield-interact-reshield pattern for private account interaction. When a user chooses the private account path, the SDK must enforce the complete pattern — the reshield step must not be skippable.

2. When using the private account path, the SDK must validate that the target account for reshielding is a private (shielded) account before submitting the transaction, and reject the operation with an explicit error if it is not.

3. The ephemeral public account created during the deshield step must never be reused across operations. Each protocol interaction from a private account must use a freshly generated account with no prior on-chain history.

### Soft Requirements

If possible.

#### Reliability

1. Multi-oracle redundancy for price feeds, with fallback when primary is stale.

2. Formal verification of auction invariants: collateral seized equals collateral in auction; debt shortfalls are correctly propagated to debt auction house.

#### Usability

1. Liquidator bot SDK with position monitoring and automated liquidation triggers.

### Out of Scope

The following are explicitly excluded from this RFP.

- Flash loan liquidations
- Liquidation aggregators (batching across multiple positions)
- Insurance funds or socialized loss mechanisms
- Multi-collateral atomic liquidations

### Privacy Architecture

All auction and liquidation state lives in public (on-chain) state. This is a deliberate architectural choice: public state enables permissionless liquidation, price feed integration, and composability.

User privacy is optionally enforced at the UX layer. The mini-app and SDK support both direct public account interaction and private account interaction via the deshield-interact-reshield pattern. When users opt to interact from a private account, the SDK must enforce the complete pattern as described below.

#### Interaction flow

For every protocol operation (liquidate, bid, settle):

1. The user initiates the action from their private account. The SDK deshields to a fresh, single-use public account with no prior on-chain history. The deshield atomically transfers both the operation token and enough native token for gas in a single indivisible action.
2. The ephemeral account executes the protocol operation.
3. The ephemeral account shields any outputs back to the user's private account. The account is never reused.

#### What is public (observable on-chain)

- All auction state: collateral amounts, bids, settlement status.
- All liquidation events and their amounts.
- Price feeds and position health data.

#### What is private

- Which private account initiated any liquidation or bid.
- Where auction proceeds go after reshielding.
- Any link between multiple protocol interactions by the same user (no on-chain linkability across ephemeral accounts).

## Platform Dependencies

### Hard blockers

These must be available on LEZ before this RFP can open.

#### Price feed

The system requires external price feeds for collateral valuation and liquidation triggers. Every hard requirement related to liquidation (F1) and reliability (R2) depends on this.

#### CDP Protocol

This liquidation system is designed to integrate with a host CDP protocol that provides positions to liquidate. It is typically deployed alongside [RFP-013](./RFP-013-reflexive-stablecoin-protocol.md) or equivalent.

#### General cross-program calls (LP-0015)

LEZ uses a tail-call execution model rather than Solana's CPI (Cross-Program Invocation). In Solana's model, a program can call another program mid-execution and resume when the call returns. In LEZ's model, a tail call hands off control entirely — there is no return.

A liquidation operation needs to: (1) call the host CDP protocol to seize collateral from an undercollateralized position, then (2) continue executing to initialize auction state and account for the seized collateral. Without general cross-program calls, step 2 cannot happen after step 1. Each continuation would need to be a separate externally callable entrypoint, which is fragile and insecure (anyone could call the continuation directly, bypassing the collateral seizure).

[LP-0015](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0015.md) (General cross-program calls via tail calls) solves this by introducing internal-only entrypoints protected by an unforgeable capability, so the liquidation program can tail-call the CDP program and have control return to a protected continuation. This prize is currently **open**.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission

Liquidator bots and indexers need to monitor positions and react to on-chain state changes. Without structured events, off-chain services must poll all accounts, which is expensive.

## Recommended Team Profile

Team experienced with:

- DeFi protocol design (liquidation systems, auction mechanisms)
- Solana or SVM program development
- Auction theory and mechanism design
- Oracle integration and price feed security
- Writing and running on-chain tests
- Front-end development for DeFi applications

## Timeline Expectations

Estimated duration: **10–12 weeks**

## Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RAI / GEB Code](https://github.com/reflexer-labs/geb)
- [RFP-001 — Admin Authority](./RFP-001-admin-authority-poc.md)
- [RFP-002 — Freeze Authority](./RFP-002-freeze-authority-poc.md)
- [RFP-008 — Lending & Borrowing](./RFP-008-lending-borrowing-protocol.md)
- [RFP-013 — Reflexive Stablecoin Protocol](./RFP-013-reflexive-stablecoin-protocol.md)
- [Appendix: Liquidation & Auction Ecosystem](../appendix/appendix-liquidation-auction-ecosystem.md)

## How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use **Discussions**.
