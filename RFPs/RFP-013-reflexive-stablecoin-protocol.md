---
id: RFP-013
title: Reflexive Stablecoin Protocol
tier: XL
status: closed
dependencies:
  - id: RFP-020
    reason: Provides the external price feed required for market price and redemption rate computation (F2, F6, R3).
  - id: LP-0013
    reason: Token transfer-authority primitives are required to custody collateral and to mint and burn the stablecoin via the LEZ Token Program (F5).
  - id: RFP-001
    reason: Provides the standardised admin authority library that gates parameter updates (stability fee, controller gains, safety ratios) per Reliability R4.
  - id: RFP-002
    reason: Provides the standardised freeze authority used for the emergency circuit breaker per Reliability R5.
  - id: LP-0015
    reason: General cross-program calls via tail calls, used to compose collateral transfers with subsequent position state updates.
  - id: LEZ-clock
    reason: Platform feature. Rate computation and stability fee accrual require elapsed time between interactions. Delivered via the LEZ clock program.
category: Applications & Integrations
---

# RFP-013 — Reflexive Stablecoin Protocol

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) for complying with any legal and regulatory requirements that apply
> to them. Software developed under the Program is published and maintained by
> its developers, not by Logos.
>
> Anyone who chooses to deploy, host, operate or use software developed under
> the Program, whether or not they were awarded a grant under the Program, does
> so at their own risk and is solely responsible for complying with any legal or
> regulatory requirements that apply to them. See the
> [Terms & Conditions](../TERMS_AND_CONDITIONS.md).
>
> Deploying the software described in this RFP, operating any service based on
> it, or carrying on business through it may amount to regulated activity in
> some jurisdictions, including where it involves holding or managing users'
> assets or providing services to others. Whoever conducts any such activity
> does so as principal, in their own name, and is solely responsible for
> assessing its regulatory treatment, including any licensing, registration,
> sanctions or anti-money laundering obligations that may apply to them. Logos
> does not make any representation, provides any advice or assumes any
> responsibility in respect of any such determination or compliance.

## Overview

Build a non-pegged, reflexive stablecoin protocol for the Logos Execution Zone
(LEZ). The stablecoin is backed by a reference token collateral and uses an
autonomous feedback controller to adjust a floating redemption price,
incentivizing market participants to stabilize the token without requiring
active governance intervention.

This design — pioneered by [RAI / Reflexer Finance](https://reflexer.finance/) —
produces a stablecoin that trades governance for mathematics: the system
self-corrects via economic incentives rather than policy decisions. For a survey
of the stablecoin ecosystem and the design rationale behind this approach, see
the
[companion appendix](../appendix/appendix-reflexive-stablecoin-ecosystem.md).
Users lock collateral and mint stablecoins against it. The protocol continuously
adjusts a redemption rate based on the deviation between market price and
redemption price. When market price exceeds redemption price, the rate goes
negative, causing redemption price to drift down and reducing demand. When
market price is below redemption price, the rate goes positive, causing
redemption price to drift up and increasing demand. This creates a
self-stabilizing feedback loop.

The protocol follows the RAI design with these key components: collateralized
debt positions (SAFEs) that track collateral, normalized debt, and accumulated
rates; a redemption price that drifts based on a computed redemption rate;
stability fees that accrue continuously via rate accumulation; and a feedback
loop that stabilizes the token mathematically rather than through governance
intervention.

## Why This Matters

The Logos ecosystem needs a credibly neutral settlement asset that reduces trust
assumptions. Traditional stablecoins are either centralized (USDC),
governance-heavy (DAI), or under-collateralized and fragile (failed algorithmic
coins). This protocol provides a fourth option: reflexive, non-pegged, and
over-collateralized.

Stablecoins enable lending, trading, and payments without the volatility of
native assets. For the private economy, this offers a stable unit of account
without requiring users to trust an issuer or governance body. Other protocols
can build on this stable asset as a risk-off collateral asset, establishing a
foundation for DeFi activity on LEZ.

## Scope of Work

### Hard Requirements

#### Functionality

1. Users can lock reference token collateral and generate stablecoin debt
   against it. All positions must maintain a minimum collateralization ratio.
   Users can add or remove collateral and repay debt subject to safety
   constraints.

2. The protocol computes a redemption rate from the deviation between market
   price and redemption price using a feedback controller. The redemption price
   drifts continuously based on this rate. Rate updates occur at bounded
   intervals with permissionless triggers.

3. Debt accrues interest continuously via a stability fee mechanism. The fee
   rate is governance-adjustable but applies uniformly to all positions.

4. Positions cannot be modified in ways that would push their collateralization
   ratio below the liquidation threshold. Read-only queries are always
   permitted.

5. The stablecoin must be a standard fungible token on LEZ, transferable and
   composable with other protocols. The protocol handles minting and burning via
   integration with the LEZ Token Program.

6. The protocol reads market prices from a configurable price feed. The
   interface requires price data with staleness detection; the specific
   implementation is flexible.

#### Usability

1. Build the program using the SPEL framework, which generates the IDL and
   client code from the program definition.

2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo.

3. Provide a CLI that covers core functionality of the program. The CLI may have
   fewer features than the GUI mini-app but must support all essential
   operations.

4. Users can view current collateralization ratio, minimum safe ratio, and
   projected outcomes of operations before executing them.

5. Users can view current redemption rate, redemption price, and projected debt
   growth from stability fees.

6. Provide an SDK that can be used to build Logos modules for interacting with
   the program. The SDK must support both public account operations and private
   account operations via the standard deshield-interact-reshield pattern.

7. Failed operations return clear, actionable error messages indicating why the
   operation was rejected.

#### Reliability

1. All financial calculations use sufficient precision to prevent overflow.
   Critical invariants include: total stablecoin minted equals sum of all
   position debts; collateralization ratios are enforced after every modifying
   operation.

2. The feedback controller must include safeguards against integral windup and
   rate explosion.

3. If the price feed is stale or unavailable, rate updates are paused. Existing
   positions remain operational but new debt generation may be restricted.

4. Parameter updates (stability fee, controller gains, safety ratios) are gated
   through admin authority.

5. Emergency circuit breaker functionality is available via freeze authority.

#### Performance

1. Position operations and rate updates must fit within LEZ transaction compute
   limits with headroom for network conditions.

2. The protocol supports thousands of concurrent positions without exceeding
   compute limits on any operation.

3. Rate updates complete within a small number of blocks. Position operations
   complete within one block.

#### Supportability

1. The program is deployed and tested on LEZ devnet/testnet.

2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   with `RISC0_DEV_MODE=0` and are included in CI.

3. CI must be green on the default branch.

4. Every hard requirement in Functionality, Usability, Reliability, and
   Performance has at least one corresponding test.

5. A README documents end-to-end usage: deployment steps, program addresses, and
   step-by-step instructions for interacting with the program via CLI and
   mini-app.

#### + Privacy

1. The mini-app and SDK must support both direct public account interaction and
   the deshield-interact-reshield pattern for private account interaction. When
   a user chooses the private account path, the SDK must enforce the complete
   pattern — the reshield step must not be skippable.

2. When using the private account path, the SDK must validate that the target
   account for reshielding is a private (shielded) account before submitting the
   transaction, and reject the operation with an explicit error if it is not.

3. The ephemeral public account created during the deshield step must never be
   reused across operations. Each protocol interaction from a private account
   must use a freshly generated account with no prior on-chain history.

### Soft Requirements

If possible.

#### Reliability

1. Multi-oracle redundancy for price feeds, with fallback when primary is stale.

2. Formal verification of critical invariants: a position with collateralization
   ratio above the minimum cannot be liquidated; total stablecoin supply equals
   total debt across all positions.

#### Usability

1. Health factor preview showing before/after state for operations.

2. Historical data display: redemption price, market price, and redemption rate
   over time.

### Out of Scope

The following are explicitly excluded from this RFP.

- Liquidation mechanism — addressed by
  [RFP-014](./RFP-014-liquidation-auction-engine.md)
- Surplus/debt management auctions — addressed by
  [RFP-014](./RFP-014-liquidation-auction-engine.md)
- Multi-collateral positions
- Private state positions (privacy via UX layer only)
- Governance token design

### Privacy Architecture

All position state lives in public (on-chain) state. This is a deliberate
architectural choice: public state enables permissionless liquidation, price
feed integration, and composability without open cryptographic research
challenges.

User privacy is optionally enforced at the UX layer. The mini-app and SDK
support both direct public account interaction and private account interaction
via the deshield-interact-reshield pattern. When users opt to interact from a
private account, the SDK must enforce the complete pattern as described below.

#### Interaction flow

For every protocol operation (lock collateral, mint, repay, withdraw):

1. The user initiates the action from their private account. The SDK deshields
   to a fresh, single-use public account with no prior on-chain history. The
   deshield atomically transfers both the operation token and enough native
   token for gas in a single indivisible action.
2. The ephemeral account executes the protocol operation.
3. The ephemeral account shields any outputs back to the user's private account.
   The account is never reused.

#### What is public (observable on-chain)

- All position state: collateral amounts, debt amounts, collateralization
  ratios.
- All protocol operations and their amounts.
- Price feeds and rate update events.

#### What is private

- Which private account funded any operation.
- Where withdrawn assets go after reshielding.
- Any link between multiple protocol interactions by the same user (no on-chain
  linkability across ephemeral accounts).

## Platform Dependencies

### Hard blockers

These must be available on LEZ before this RFP can open.

#### Price feed

The protocol requires external price feeds for market price and redemption rate
computation. Every hard requirement related to rate adjustment (F2) and
reliability (R3) depends on this.
[RFP-020](./RFP-020-redstone-oracle-adaptor.md) (RedStone off-chain oracle
adaptor) delivers external price feeds; [RFP-019](./RFP-019-twap-oracle.md)
(on-chain TWAP oracle) is an alternative once a DEX is live.

#### Token authorities (LP-0013)

The stablecoin program is a token custodian: it holds locked collateral per
position and mints and burns the stablecoin via the LEZ Token Program (F5). This
requires the transfer-authority primitives in
[LP-0013](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md),
currently **open**.

#### Authority libraries (RFP-001, RFP-002)

Parameter updates are gated through an admin authority (R4) and the emergency
circuit breaker uses a freeze authority (R5). These come from the standardised
libraries in [RFP-001](./RFP-001-admin-authority-lib.md) and
[RFP-002](./RFP-002-freeze-authority-lib.md). Both RFPs are closed (candidate
picked) and the libraries are in development.

### Resolved dependencies

These primitives were once blockers but are now delivered on LEZ, so they no
longer gate this RFP. They remain in the frontmatter `dependencies` index for
traceability.

#### On-chain clock / timestamp

Rate computation and stability fee accrual require knowing how much time has
elapsed between interactions. The LEZ clock program now maintains on-chain
timestamp accounts that any program can read, so this is **delivered**.

#### General cross-program calls (LP-0015)

A stablecoin operation like "lock collateral and mint" needs to: (1) call the
token program to transfer collateral into the position, then (2) continue
executing to update position state and mint stablecoins. General cross-program
calls via tail calls let step 2 run in a protected continuation after step 1.
[LP-0015](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0015.md)
is **closed**, delivered by the LEZ team as part of the core runtime.

#### Event emission (LP-0012)

Off-chain services observe positions and rate updates through structured events
rather than polling every account.
[LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)
is **closed**.

### Soft blockers

Desirable but the RFP can open without them.

#### Liquidation engine (RFP-014)

A full liquidation and auction engine is delivered by
[RFP-014](./RFP-014-liquidation-auction-engine.md), which liquidates the
positions this protocol creates. The stablecoin is more robust once RFP-014 is
live, but it can ship with a placeholder liquidation path that satisfies its
functionality requirements and integrate RFP-014 once available. It is therefore
a soft dependency rather than a frontmatter entry (also avoiding a hard cycle,
since RFP-014 depends on this RFP for the positions it liquidates).

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
- [RFP-001 — Admin Authority](./RFP-001-admin-authority-lib.md)
- [RFP-002 — Freeze Authority](./RFP-002-freeze-authority-lib.md)
- [RFP-008 — Lending & Borrowing](./RFP-008-lending-borrowing-protocol.md)
- [RFP-014 — Liquidation & Auction Engine](./RFP-014-liquidation-auction-engine.md)
- [Appendix: Reflexive Stablecoin Ecosystem](../appendix/appendix-reflexive-stablecoin-ecosystem.md)

## How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
