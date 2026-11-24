---
id: RFP-013
title: Reflexive Stablecoin Protocol
tier: XL
status: closed
dependencies:
  - id: RFP-020
    reason: Provides the external price feed required for market price and redemption rate computation (F2, F6, R3).
  - id: LEZ-token-program
    reason: Requires a token program compatible with the LEZ testnet 0.4 actor model, with messages for collateral custody, minting and burning authorised by stablecoin-controlled PDAs (F5).
  - id: RFP-001
    reason: Provides the standardised admin authority library that gates parameter updates (stability fee, controller gains, safety ratios) per Reliability R4.
  - id: RFP-002
    reason: Provides the standardised freeze authority used for the emergency circuit breaker per Reliability R5.
  - id: LEZ-actor-messaging
    reason: Synchronous and asynchronous actor messages compose collateral transfers and position updates, with authenticated internal messages and explicit PDA authority.
  - id: LEZ-clock
    reason: Platform feature. Rate computation and stability fee accrual require elapsed time between interactions. Delivered via the LEZ clock program.
category: Applications & Integrations
---

# RFP-013 — Reflexive Stablecoin Protocol

> **Combined proposals.** Proposals covering both this RFP and
> [RFP-033](./RFP-033-stablecoin-ui.md), its position-management interface, will
> be given preference. The UI is a separate scope, not a soft requirement of
> this RFP.

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) complying with any legal and regulatory requirements that apply to
> them. Software developed under the Program is published and maintained by its
> developers, not by Logos.
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
> does not make any representation, provide any advice or assume any
> responsibility in respect of any such determination or compliance.

## Overview

Build a non-pegged, reflexive stablecoin protocol for the Logos Execution Zone
(LEZ). The stablecoin is backed by a reference token collateral and uses an
autonomous feedback controller to adjust a floating redemption price,
incentivizing market participants to stabilize the token without requiring
active governance intervention.

The stablecoin LEZ program is delivered for LEZ testnet 0.4 (Supportability
requirement #1).

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

5. The stablecoin must be a fungible token on LEZ, transferable and composable
   with other protocols. Minting, burning and collateral custody use messages to
   the token program from PDAs the stablecoin program controls.

6. The protocol reads market prices from a configurable price feed. The
   interface requires price data with staleness detection; the specific
   implementation is flexible. The feed exposes read or assertion messages
   compatible with the LEZ 0.4 actor model.

7. The program answers read requests or assertions for position and protocol
   state from other programs, so RFP-014 and other integrators compose without
   reading its state directly. Replies are authenticated against the expected
   actor.

8. Token, price-feed and clock program addresses are configured for each
   deployment rather than hardcoded. PDAs derive from the stablecoin program
   deployment address and documented seeds.

#### Usability

1. Build the program for the LEZ 0.4 actor model. Use the SPEL framework to
   generate the IDL and client code if it supports that model at delivery;
   otherwise publish an equivalent IDL covering each message, its permitted
   senders, its target actor, the state types and events.

2. Provide a CLI using the Logos Core headless framework and the core module
   below. It covers opening positions, depositing and withdrawing collateral,
   minting, partial and full repayment, closing, listing positions and recovery
   on both public and private account paths.

3. Expose current collateralization ratio, minimum safe ratio, projected
   before/after outcomes, maximum withdrawable collateral and mintable debt, and
   the redemption price at which a position reaches the minimum ratio through
   the core module and CLI. Calculations use the program's arithmetic and
   rounding and identify the time and chain-state snapshot they describe.

4. Expose current redemption rate, redemption price, accrued stability fees, and
   projected debt and collateralization ratio over a caller-selected horizon
   through the core module and CLI. Projections state their assumptions,
   including unchanged redemption and stability fee rates.

5. Provide a Logos core module for program operations and shared client
   functionality, reusable by the CLI and QML ui modules. It supports public and
   private account operations, including private funding and output return,
   position discovery, recovery and the calculations above. It uses the LEZ core
   module wallet for signing and key access. No separate SDK is required.

6. The core module and CLI simulate each operation before submission and do not
   submit an operation that simulation rejects. Simulation does not guarantee
   success if state changes before inclusion. Failed operations, including
   reverted transactions that were included and charged, return clear,
   actionable error messages indicating why the operation was rejected.

7. Derive position-owner accounts and nonces deterministically from the user's
   wallet seed through the wallet, taking the stablecoin program's deployed
   address as an input. Document how the deployment address and configuration
   are rediscovered from chain data during seed recovery. Document the
   derivation scheme, versioning, discovery algorithm and scan termination rule,
   including handling of closed positions, unused derivation gaps and
   interrupted creation. Nonces must not be reused after closure. All positions
   created through the core module must be recoverable on a fresh install from
   the seed and chain data alone, without local cached state or a separately
   backed-up secret. The CLI consumer uses the same scheme. Include recovery
   tests with at least 20 positions, including closed positions, gaps and
   interrupted operations.

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

1. Position operations and rate updates must fit within the LEZ testnet 0.4
   transaction gas limit with headroom for network conditions. Document the gas
   cost of each operation. Measurements are taken on LEZ testnet 0.4.

2. The protocol supports thousands of concurrent positions without exceeding the
   LEZ testnet 0.4 transaction gas limit on any operation.

3. Rate updates complete within a small number of blocks. Position operations
   complete within one block.

#### Supportability

1. The stablecoin LEZ program is delivered for LEZ testnet 0.4: it is deployed
   and tested on LEZ testnet 0.4.

2. End-to-end integration tests run against a LEZ sequencer (standalone mode)
   with `RISC0_DEV_MODE=0` and are included in CI.

3. CI must be green on the default branch.

4. Every hard requirement in Functionality, Usability, Reliability, and
   Performance and Privacy has at least one corresponding test.

5. A README documents end-to-end usage: deployment steps, program addresses, and
   step-by-step CLI instructions. The core module's API is documented for UI
   integrators.

#### + Privacy

1. The core module and CLI support public and private account interaction. On
   the private path, collateral and fees are funded from the user's private
   account, and minted stablecoin and withdrawn collateral end in a private
   account of the user. The module must not offer a completion path that leaves
   these outputs public.

2. The module validates that output return targets are private accounts. Outputs
   delivered to a private account as stored messages are collected by the
   module, including after interruption. After interruption, it detects
   incomplete operations and supports completing them or returning funds to the
   user's private account. Recovery state must be reconstructible from the seed
   and chain data.

3. On the private path, use one public owner account per position, retained for
   subsequent operations on that position. Funding and fee-paying accounts must
   not be shared across positions or with the user's public account activity. No
   ownership, funding or fee-payment relationship recorded on-chain links these
   accounts to the user's private account, other positions or public accounts.
   Document the linkage of operations on the same position and residual amount,
   timing and node-query correlations; this does not eliminate those inferences.

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
- The position-management mini-app. The core module and CLI remain in scope.
- Multi-collateral positions
- Private state positions (privacy via UX layer only)
- Governance token design

### Privacy Architecture

All position state lives in public (on-chain) state. This is a deliberate
architectural choice: public state enables permissionless liquidation, price
feed integration, and composability without open cryptographic research
challenges.

The core module supports both public and private account paths. UI consumers use
its private funding, signing, discovery and output-return capabilities rather
than implementing a second account lifecycle.

#### Interaction flow

For each private-path position operation, the module funds the operation and
fees from a private account, executes the operation using the position's owner
where required, and returns outputs to a private account. The position owner
persists for that position only. Funding and fee-payment arrangements must meet
the Privacy requirements above; the implementation mechanism is left to the
proposer. Interrupted operations remain recoverable through the same module.

#### What is public (observable on-chain)

- All position state: collateral amounts, debt amounts, collateralization
  ratios.
- All protocol operations and their amounts, including links between operations
  on the same position.
- Price feeds and rate update events.

#### Privacy boundaries

The private path must not record ownership, funding or fee-payment links to the
user's private account, other positions or public accounts. Public amounts and
timing may still be correlated, and a configured node may correlate queried
accounts. Document these residual correlations and their mitigations. Seed
recovery does not make position state private.

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

#### Token program for LEZ testnet 0.4

A token program compatible with the LEZ 0.4 actor model must provide messages
for transfers, collateral custody, minting and burning, with authority granted
by PDAs controlled by the stablecoin program. A pre-0.4 transfer-authority
implementation alone does not satisfy this dependency. Proposals must identify
the token implementation, its deployed address and supported message interface,
and account for any required port before end-to-end acceptance.

#### Authority libraries (RFP-001, RFP-002)

Parameter updates are gated through an admin authority (R4) and the emergency
circuit breaker uses a freeze authority (R5). These come from the standardised
libraries in [RFP-001](./RFP-001-admin-authority-lib.md) and
[RFP-002](./RFP-002-freeze-authority-lib.md). Both RFPs are closed (candidate
picked) and the libraries are in development.

The price-feed and authority interfaces must support LEZ 0.4 actor messaging.
Proposals must identify the implementations and interface versions they use;
pre-0.4 SPEL extensions cannot be assumed compatible without adaptation.

### Runtime dependencies

#### On-chain clock / timestamp

Rate computation and stability fee accrual require elapsed time. Obtain or
validate timestamps through messages to the configured LEZ clock actor, not by
reading another program's state. Proposals must document the clock message
interface used, including timestamp assertions where applicable.

#### Actor messaging

Operations such as locking collateral and minting compose synchronous and
asynchronous messages instead of tail calls. Internal-only messages must check
`from_own_program()`; external replies must be authenticated against the
expected actor. Synchronous calls requiring PDA authority carry the appropriate
seed grants; asynchronous messages must not assume that grant is propagated.
Clients handle deferred credits and collect stored messages before reporting
private-path output delivery as complete.

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
- Logos core module and CLI development

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
