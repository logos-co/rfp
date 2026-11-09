---
note: This metadata is for internal programme tracking only. It is not part of this specification and does not form part of its requirements.
id: RFP-013
title: Reflexive Stablecoin Protocol
tier: XL
status: closed
dependencies:
  - id: RFP-020
    reason: External price feeds are required for market price and redemption rate computation (F2, F6, R3); RFP-020 is an illustrative example.
  - id: LP-0013
    reason: Token transfer authorities are required for the program to hold locked collateral and to mint and burn the stablecoin via the LEZ Token Program (F5); LP-0013 is an illustrative example.
  - id: RFP-001
    reason: An admin authority capability is required to gate parameter updates (stability fee, controller gains, safety ratios) per R4 and F7; RFP-001 describes such a capability and is an illustrative example.
  - id: RFP-002
    reason: A freeze authority capability is required for the emergency circuit breaker per R5 and F7; RFP-002 describes such a capability and is an illustrative example.
  - id: LP-0015
    reason: General cross-program calls via tail calls, used to compose collateral transfers with subsequent position state updates; available in the LEZ core runtime.
  - id: LEZ-clock
    reason: Platform feature. Rate computation and stability fee accrual require elapsed time between interactions; available via the LEZ clock program.
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
autonomous feedback controller to adjust a floating redemption price. The
program may be independently deployed and configured by those who choose to
deploy it (F7).

The program targets LEZ testnet 0.4 (Supportability requirement #1).

The design follows the model deployed by
[RAI / Reflexer Finance](https://reflexer.finance/) on Ethereum, in which a
controller computes the redemption rate from market data instead of the rate
being set by governance votes. For a survey of the stablecoin ecosystem and the
design rationale behind this approach, see the
[companion appendix](../appendix/appendix-reflexive-stablecoin-ecosystem.md).
Users lock collateral and mint stablecoins against it. The protocol continuously
adjusts a redemption rate based on the deviation between market price and
redemption price. When market price exceeds redemption price, the rate goes
negative and the redemption price drifts down. When market price is below
redemption price, the rate goes positive and the redemption price drifts up. The
redemption price, in turn, changes the incentive for market participants to
mint, repay or hold the token.

The protocol follows the RAI design with these key components: collateralized
debt positions (SAFEs) that track collateral, normalized debt, and accumulated
rates; a redemption price that drifts based on a computed redemption rate;
stability fees that accrue continuously via rate accumulation; and a feedback
controller that sets the redemption rate from market data rather than by
governance decision.

## Why This Matters

Stablecoin designs differ in how their value is referenced and who adjusts their
parameters. Fiat-backed stablecoins such as USDC rely on reserves held by an
issuer. CDP stablecoins such as Sky/MakerDAO's DAI are minted against
collateral, with stability fees, collateral types and debt ceilings set by token
governance votes. In the reflexive model used by RAI, and by its fork HAI on
Optimism, a feedback controller computes the redemption rate from the deviation
between market price and redemption price, and the parameters left to an admin
authority are limited (see the
[companion appendix](../appendix/appendix-reflexive-stablecoin-ecosystem.md)).

This RFP specifies that model as a LEZ program. The stablecoin it issues is a
standard fungible LEZ token (F5), so other LEZ programs can accept it as they
would any other token. Position state is public; users may interact from a
private account through the deshield-interact-reshield pattern (see Privacy
Architecture).

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
   rate is adjustable through the admin authority (F7, R4) and applies uniformly
   to all positions.

4. Positions cannot be modified in ways that would push their collateralization
   ratio below the liquidation threshold. Read-only queries are always
   permitted.

5. The stablecoin must be a standard fungible token on LEZ, transferable and
   composable with other protocols. The protocol handles minting and burning via
   integration with the LEZ Token Program.

6. The protocol reads market prices from a configurable price feed. The
   interface requires price data with staleness detection; the specific
   implementation is flexible, and the price feed used is a configuration choice
   of the instance (F7).

7. The admin authority (stability fee, controller gains, safety ratios; R4) and
   the freeze authority (emergency circuit breaker; R5) of each instance, its
   price feed (F6), and its parameter values are set when the instance is
   initialised. The program contains no hardcoded authority holder and no
   hardcoded production parameter values; these are configuration choices of
   whoever initialises the instance.

#### Usability

1. Build the program using the SPEL framework, which generates the IDL and
   client code from the program definition.

2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via the team's own git repo.

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
   through an admin authority (F7).

5. Emergency circuit breaker functionality is available via a freeze authority
   (F7).

#### Performance

1. Position operations and rate updates must fit within LEZ transaction compute
   limits with headroom for network conditions.

2. The protocol supports thousands of concurrent positions without exceeding
   compute limits on any operation.

3. Rate updates complete within a small number of blocks. Position operations
   complete within one block.

#### Supportability

1. The team deploys and tests the program on LEZ testnet 0.4.

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
   a user chooses the private account path, the SDK must complete the full
   deshield-interact-reshield pattern for the operation.

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

2. Formal verification of critical invariants, with the verification reports
   published alongside the codebase: a position with collateralization ratio
   above the minimum cannot be liquidated; total stablecoin supply equals total
   debt across all positions.

#### Usability

1. Health factor preview showing before/after state for operations.

2. Historical data display: redemption price, market price, and redemption rate
   over time.

### Out of Scope

The following are explicitly excluded from this RFP.

- Liquidation mechanism (specified separately in
  [RFP-014](./RFP-014-liquidation-auction-engine.md))
- Surplus and debt management auctions (specified separately in
  [RFP-014](./RFP-014-liquidation-auction-engine.md))
- Multi-collateral positions
- Private state positions (privacy via UX layer only)
- Governance token design
- Mainnet deployment. Mainnet deployment is not part of this RFP: whether, when
  and by whom the software is deployed is a matter for those who deploy it.
- The identity of persons holding the admin and freeze authorities, and the
  exercise of those authorities, including the choice of price feeds and
  parameter values, in any deployment, are matters for the relevant independent
  deployer or operator and are outside the scope of this RFP.

### Privacy Architecture

All position state lives in public (on-chain) state. This is a deliberate
architectural choice: public state enables permissionless liquidation, price
feed integration, and composability without open cryptographic research
challenges.

User privacy is optionally enforced at the UX layer. The mini-app and SDK
support both direct public account interaction and private account interaction
via the deshield-interact-reshield pattern. When users opt to interact from a
private account, the SDK must complete the full pattern as described below.

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

#### What is not recorded in public on-chain state

- The private account that funded an operation.
- The destination of withdrawn assets after reshielding.
- Any association between the ephemeral accounts used by the same private
  account. Timing and amounts of operations remain public and may still be
  correlated.

## Platform Dependencies

Implementation of the stablecoin program depends on the availability, on testnet
0.4, of the following technical capabilities:

1. **External price feeds** for market price and redemption rate computation.
   Every hard requirement related to rate adjustment (F2) and reliability (R3)
   depends on this. The price feed used is a configuration choice of the
   instance (F6, F7). [RFP-020](./RFP-020-redstone-oracle-adaptor.md) and
   [RFP-019](./RFP-019-twap-oracle.md) are illustrative examples.
2. **Token transfer authorities**, so that the program can hold locked
   collateral per position and mint and burn the stablecoin via the LEZ Token
   Program according to its programmed rules (F5).
   [LP-0013](https://github.com/logos-co/lambda-prize/blob/master/prizes/LP-0013.md)
   is an illustrative example.
3. **Admin and freeze authority capabilities**, used to gate parameter updates
   and to provide the emergency circuit breaker (R4, R5, F7). These capabilities
   are described in [RFP-001](./RFP-001-admin-authority-lib.md) and
   [RFP-002](./RFP-002-freeze-authority-lib.md), which are illustrative
   examples.

The following capabilities are already available in the LEZ core runtime:

- **An on-chain clock**, so that rate computation and stability fee accrual can
  read the time elapsed between interactions from the timestamp accounts
  maintained by the LEZ clock program.
- **General cross-program calls via tail calls**, so that an operation such as
  "lock collateral and mint" can call the token program to transfer collateral
  into the position and then continue, in a protected continuation, to update
  position state and mint stablecoins
  ([LP-0015](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0015.md)).
- **Structured event emission**, so that off-chain services can observe
  positions and rate updates without polling every account
  ([LP-0012](https://github.com/logos-co/lambda-prize/blob/main/prizes/LP-0012.md)).

Liquidation is outside the scope of this RFP. The positions this protocol
creates are liquidated by a separate liquidation and auction program;
[RFP-014](./RFP-014-liquidation-auction-engine.md) specifies such a program,
with a host CDP interface, and is an illustrative example. Until a liquidation
program is integrated, the protocol may use a placeholder liquidation path that
satisfies its functionality requirements. This capability is not listed in the
frontmatter, since RFP-014 lists this RFP as its host CDP example.

Proposals must state which implementation of each capability they will build
against, and how a change to that implementation's interface would be absorbed.

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

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
