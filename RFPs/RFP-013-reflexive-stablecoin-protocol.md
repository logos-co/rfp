---
id: RFP-013
title: Reflexive Stablecoin Protocol
tier: XL
funding: $XXXXX
status: draft
dependencies: RFP-001 (Admin Authority), RFP-002 (Freeze Authority)
category: Applications & Integrations
---

# RFP-013 — Reflexive Stablecoin Protocol

## Overview

Build a non-pegged, reflexive stablecoin protocol for the Logos Execution Zone (LEZ). The stablecoin is backed by a reference token collateral and uses an autonomous feedback controller to adjust a floating redemption price, incentivizing market participants to stabilize the token without requiring active governance intervention.

This design — pioneered by [RAI / Reflexer Finance](https://reflexer.finance/) — produces a stablecoin that trades governance for mathematics: the system self-corrects via economic incentives rather than policy decisions. Users lock collateral and mint stablecoins against it. The protocol continuously adjusts a redemption rate based on the deviation between market price and redemption price. When market price exceeds redemption price, the rate goes negative, causing redemption price to drift down and reducing demand. When market price is below redemption price, the rate goes positive, causing redemption price to drift up and increasing demand. This creates a self-stabilizing feedback loop.

The protocol follows the RAI design with these key components: collateralized debt positions (SAFEs) that track collateral, normalized debt, and accumulated rates; a redemption price that drifts based on a computed redemption rate; stability fees that accrue continuously via rate accumulation; and a feedback loop that stabilizes the token mathematically rather than through governance intervention.

## Why This Matters

The Logos ecosystem needs a credibly neutral settlement asset that reduces trust assumptions. Traditional stablecoins are either centralized (USDC), governance-heavy (DAI), or under-collateralized and fragile (failed algorithmic coins). This protocol provides a fourth option: reflexive, non-pegged, and over-collateralized.

Stablecoins enable lending, trading, and payments without the volatility of native assets. For the private economy, this offers a stable unit of account without requiring users to trust an issuer or governance body. Other protocols can build on this stable asset as a risk-off collateral asset, establishing a foundation for DeFi activity on LEZ.

## Scope of Work

### Hard Requirements

#### Functionality

1. Users can lock reference token collateral and generate stablecoin debt against it. All positions must maintain a minimum collateralization ratio. Users can add or remove collateral and repay debt subject to safety constraints.

2. The protocol computes a redemption rate from the deviation between market price and redemption price using a feedback controller. The redemption price drifts continuously based on this rate. Rate updates occur at bounded intervals with permissionless triggers.

3. Debt accrues interest continuously via a stability fee mechanism. The fee rate is governance-adjustable but applies uniformly to all positions.

4. Positions cannot be modified in ways that would push their collateralization ratio below the liquidation threshold. Read-only queries are always permitted.

5. The stablecoin must be a standard fungible token on LEZ, transferable and composable with other protocols. The protocol handles minting and burning via integration with the LEZ Token Program.

6. The protocol reads market prices from a configurable price feed. The interface requires price data with staleness detection; the specific implementation is flexible.

#### Usability

1. Build the program using the SPEL framework, which generates the IDL and client code from the program definition.

2. Provide a Logos mini-app GUI with local build instructions, downloadable assets, and loadable in Logos app (Basecamp) via git repo.

3. Provide a CLI that covers core functionality of the program. The CLI may have fewer features than the GUI mini-app but must support all essential operations.

4. Users can view current collateralization ratio, minimum safe ratio, and projected outcomes of operations before executing them.

5. Users can view current redemption rate, redemption price, and projected debt growth from stability fees.

6. The SDK supports both public account operations and private account operations via the standard deshield-interact-reshield pattern.

7. Failed operations return clear, actionable error messages indicating why the operation was rejected.

#### Reliability

1. All financial calculations use sufficient precision to prevent overflow. Critical invariants include: total stablecoin minted equals sum of all position debts; collateralization ratios are enforced after every modifying operation.

2. The feedback controller must include safeguards against integral windup and rate explosion.

3. If the price feed is stale or unavailable, rate updates are paused. Existing positions remain operational but new debt generation may be restricted.

4. Parameter updates (stability fee, controller gains, safety ratios) are gated through admin authority. Emergency circuit breaker functionality is available via freeze authority.

#### Performance

1. Position operations and rate updates must fit within LEZ transaction compute limits with headroom for network conditions.

2. The protocol supports thousands of concurrent positions without exceeding compute limits on any operation.

3. Rate updates complete within a small number of blocks. Position operations complete within one block.

#### Supportability

1. The program is deployed and tested on LEZ devnet/testnet.

2. End-to-end integration tests run against a LEZ sequencer (standalone mode) and are included in CI.

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

2. Formal verification of critical invariants: a position with collateralization ratio above the minimum cannot be liquidated; total stablecoin supply equals total debt across all positions.

#### Usability

1. Health factor preview showing before/after state for operations.

2. Historical data display: redemption price, market price, and redemption rate over time.

### Out of Scope

The following are explicitly excluded from this RFP.

- Liquidation mechanism — addressed by [RFP-014](./RFP-014-liquidation-auction-engine.md)
- Surplus/debt management auctions — addressed by [RFP-014](./RFP-014-liquidation-auction-engine.md)
- Multi-collateral positions
- Private state positions (privacy via UX layer only)
- Governance token design

### Privacy Architecture

All position state lives in public (on-chain) state. This is a deliberate architectural choice: public state enables permissionless liquidation, price feed integration, and composability without open cryptographic research challenges.

User privacy is optionally enforced at the UX layer. The mini-app and SDK support both direct public account interaction and private account interaction via the deshield-interact-reshield pattern. When users opt to interact from a private account, the SDK must enforce the complete pattern as described below.

#### Interaction flow

For every protocol operation (lock collateral, mint, repay, withdraw):

1. The user initiates the action from their private account. The SDK deshields to a fresh, single-use public account with no prior on-chain history. The deshield atomically transfers both the operation token and enough native token for gas in a single indivisible action.
2. The ephemeral account executes the protocol operation.
3. The ephemeral account shields any outputs back to the user's private account. The account is never reused.

#### What is public (observable on-chain)

- All position state: collateral amounts, debt amounts, collateralization ratios.
- All protocol operations and their amounts.
- Price feeds and rate update events.

#### What is private

- Which private account funded any operation.
- Where withdrawn assets go after reshielding.
- Any link between multiple protocol interactions by the same user (no on-chain linkability across ephemeral accounts).

## Platform Dependencies

### Hard blockers

These must be available on LEZ before this RFP can open.

#### Price feed

The protocol requires external price feeds for market price and redemption rate computation. Every hard requirement related to rate adjustment (F2) and reliability (R3) depends on this.

#### On-chain clock / timestamp

Rate computation and stability fee accrual require knowing how much time has elapsed between interactions. Without a reliable on-chain timestamp, these cannot be computed.

### Soft blockers

Desirable but the RFP can open without them.

#### Event emission

Off-chain services need to monitor positions and rate updates. Without structured events, services must poll all accounts, which is expensive.

## Recommended Team Profile

Team experienced with:

- DeFi protocol design (CDP systems, RAI/Reflexer patterns)
- Solana or SVM program development
- Fixed-point arithmetic and numerical methods
- Interest rate modelling and control theory
- Oracle integration and price feed security
- Writing and running on-chain tests
- Front-end development for DeFi applications

## Timeline Expectations

Estimated duration: **14–18 weeks**

## Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RAI Whitepaper](https://github.com/reflexer-labs/whitepapers/blob/master/English/rai-english.pdf)
- [RFP-001 — Admin Authority](./RFP-001-admin-authority-poc.md)
- [RFP-002 — Freeze Authority](./RFP-002-freeze-authority-poc.md)
- [RFP-008 — Lending & Borrowing](./RFP-008-lending-borrowing-protocol.md)
- [RFP-014 — Liquidation & Auction Engine](./RFP-014-liquidation-auction-engine.md)

## How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use **Discussions**.
