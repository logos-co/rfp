---
note: This metadata is for internal programme tracking only. It is not part of this specification and does not form part of its requirements.
id: RFP-024
title: CDP-Backed Synthetic Assets
tier: L
status: draft
category: Applications & Integrations
dependencies: []
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-024 — CDP-Backed Synthetic Assets

> **Status:** Draft. This RFP is not yet accepting proposal submissions.

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

## 🧭 Overview

Build a collateralised debt position (CDP) synthetic-asset protocol on LEZ.
Users lock collateral in a position and mint a synthetic token (sASSET) that
tracks the price of another asset, such as `sETH`, `sBTC`, `sXMR` or `sZEC`, as
reported by a price feed. Burning sASSET repays the position's debt and releases
its collateral. Positions that fall below their liquidation threshold are
liquidated against a stability pool of sASSET deposits, in the style of Liquity,
so that liquidation does not depend on buying sASSET on a secondary market. One
program supports several synthetics, each configured with its own price feed and
risk parameters, and may be independently deployed and configured by those who
choose to deploy it.

The program is written for the LEZ v0.4 actor model and the team deploys and
tests it on LEZ testnet 0.4 (Supportability #1). The applying team should have
experience with CDP protocol design (Liquity, MakerDAO, Synthetix or similar)
and with integrating price oracles.

## 🔥 Why This Matters

A synthetic asset gives LEZ programs price exposure to an asset that does not
exist on LEZ, without bridging that asset or holding it: the program holds only
LEZ-side collateral, and the synthetic is a debt position against it. Once
minted, sASSET is a fungible LEZ token that DEXes, lending markets and other LEZ
programs can integrate like any other token.

Synthetix used this structure to list synthetics for several assets (including
`sETH`, `sBTC` and `sXMR`) from a single CDP-minting engine, adding each asset
through a price feed and parameters rather than a new protocol. Liquity
introduced the stability pool, which absorbs liquidated debt from pre-deposited
tokens within the liquidation transaction.

## 🏗 Design Rationale

### Liquidation in a thin secondary market

In a keeper-based CDP, a liquidator buys the synthetic on the market, repays the
position's debt and claims the collateral at a discount. For a synthetic that
tracks a volatile asset and has a thin secondary market, the synthetic the
liquidator needs can cost more than the discount by the time it is bought. If no
liquidator acts, the position's debt exceeds its collateral value and the system
carries bad debt. The faster the tracked asset moves, the narrower the range in
which keeper liquidation is profitable.

A stability pool removes that dependence: the debt is burned from pre-deposited
sASSET within the liquidation transaction, and the seized collateral is credited
to pool depositors. The trigger becomes a call any account can make. Because
pool deposits and outstanding debt are both on-chain, issuance can be capped
relative to pool deposits. The stability pool is the required baseline
(Functionality #12 to #17); proposals may describe an alternative absorption
mechanism if they show how it handles the case above (Functionality #18).

### One engine, many synthetics

The mint, repay and liquidate logic is the same whichever asset is tracked; what
varies per synthetic is the price feed, the risk parameters and the stability
pool. Adding a synthetic is therefore configuration, not a program change
(Functionality #2). The quality of each synthetic's price feed (redundancy,
liquidity of the underlying markets, manipulation resistance) is the main
per-asset risk, which is why feed guards (Functionality #10) and a per-feed
assessment (Supportability #12) are required.

## ✅ Scope of Work

### Hard Requirements

#### Functionality

##### Synthetics and positions

1. One instance of the program supports multiple synthetics. Each synthetic is a
   fungible LEZ token (sASSET) tracking the price of one asset, as reported by
   the synthetic's configured price feeds.
2. A synthetic is added through the admin authority by configuring its price
   feeds and parameters (Functionality #19), with no change to the program.
3. A user opens a position by depositing collateral from the instance's
   collateral allow-list and minting sASSET, provided the position's
   collateralisation ratio (c-ratio) at the current price is at or above the
   synthetic's minimum c-ratio.
4. A position holder can add collateral, withdraw collateral, mint further
   sASSET and repay sASSET in any amount. Withdrawals and mints are rejected if
   they would leave the position below the minimum c-ratio. Repaying the full
   debt releases all remaining collateral.
5. Only the position holder can withdraw collateral, mint against a position or
   close it. Anyone can repay debt or add collateral to a position.
6. Any fees configured for the instance (mint, repay or stability fees) accrue
   to the configured fee recipient. Fees may be set to zero.
7. The proposal specifies its debt model (debt shared across synthetics, debt
   shared within a pool, or isolated per-position debt) and documents its
   failure modes, including how a price move in one synthetic affects positions
   in other synthetics that share its debt.
8. Each synthetic has a configurable debt ceiling, and each collateral asset a
   configurable concentration cap. Mints that would exceed either are rejected.

##### Price feeds

09. The program obtains prices from the configured price-feed programs by
    message. Price data follows the canonical oracle price data standard
    specified in [RFP-019](./RFP-019-twap-oracle.md) (asset identifiers, price,
    confidence, timestamp, source).
10. Each synthetic can be configured with several price feeds, combined by
    median, with a maximum price deviation between feeds and a maximum price
    age. Mints and collateral withdrawals are rejected when the price fails
    either guard. The proposal specifies and documents how liquidation behaves
    in that case.
11. A TWAP of a LEZ pool trading an sASSET is not accepted as a price feed for
    that sASSET.

##### Liquidation

12. Any account can trigger the liquidation of a position whose c-ratio is below
    the synthetic's liquidation threshold. Liquidation does not depend on any
    particular operator.
13. Each synthetic has a stability pool. Any account can deposit sASSET into it
    and withdraw its deposit, less its share of debt absorbed by liquidations.
14. On liquidation, the program burns sASSET from the stability pool equal to
    the position's debt, credits the position's collateral (up to the debt's
    value plus the configured liquidation penalty) to pool depositors pro rata
    to their deposits, and returns any remaining collateral to the position
    holder.
15. Burning from the pool, writing off the position's debt and crediting the
    collateral happen in the same transaction and succeed or fail together.
16. The proposal specifies what happens when the stability pool cannot cover a
    position's debt in full (for example redistribution of the remaining debt
    and collateral to other positions, as in Liquity, or a keeper fallback) and
    implements it.
17. Each synthetic can be configured with an issuance cap expressed relative to
    its stability pool deposits. Mints that would exceed it are rejected.
18. A proposal that uses a liquidation absorption mechanism other than a
    stability pool must show how it handles the thin-secondary-market case in
    the Design Rationale, and meet Functionality #12, #15 and #16 with it.

##### Configuration and roles

19. The following are set when an instance is initialised and, where the
    instance allows it, changed through its admin authority:
    - the collateral allow-list and concentration caps;
    - the synthetics, and for each one its price feeds, minimum c-ratio,
      liquidation threshold, liquidation penalty, debt ceiling, issuance cap and
      price guards;
    - any fees and the fee recipient;
    - the deployed addresses of the price-feed, collateral-token and sASSET
      token programs the instance uses;
    - the admin and freeze authorities.
20. No authority holder, fee recipient, price feed or production parameter value
    is hardcoded.
21. The freeze authority can pause minting and collateral withdrawal, per
    synthetic or for the whole instance. Repayment, adding collateral,
    liquidation and stability pool withdrawals remain available while paused.
22. The program emits events for position changes (open, mint, repay, collateral
    changes, close), liquidations, stability pool deposits and withdrawals, and
    configuration changes.

#### Usability

1. Provide an SDK that can be used to build Logos modules for interacting with
   the program.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via the team's own git repo.
3. Provide a CLI that covers core functionality of the program. The CLI may have
   fewer features than the GUI mini-app but must support all essential
   operations, including triggering a liquidation.
4. Provide an IDL for the LEZ program.
5. The GUI displays, for each of the user's positions: collateral, debt,
   c-ratio, liquidation price, and the current price with its age.
6. The GUI displays, for each synthetic: total debt, debt ceiling, issuance cap
   headroom and stability pool deposits.
7. The GUI displays, for each of the user's stability pool deposits: the current
   deposit and the collateral gained from liquidations.
8. sASSET and the program expose a documented message interface, so that DEXes,
   lending markets and other LEZ programs can integrate sASSET without changes
   to this program.

#### Reliability

1. For each synthetic, outstanding sASSET supply equals the total debt recorded
   across positions, except for debt redistributed under Functionality #16,
   which is accounted for separately.
2. No user operation other than a price change leaves a position below the
   minimum c-ratio.
3. Stability pool accounting is exact: the sum of depositors' withdrawable
   deposits and collateral gains equals the pool's holdings, up to documented
   rounding.
4. Mint, repay and liquidation flows do not depend on a credit to a private
   account being delivered in the same transaction.

#### Performance

1. Document the compute usage of each operation (open, mint, repay, collateral
   changes, liquidation, stability pool deposit and withdrawal). LEZ's
   per-transaction compute budget may change during testnet.
2. The cost of liquidating a position does not grow with the number of stability
   pool depositors.

#### Supportability

01. The team deploys and tests the program on LEZ testnet 0.4, with at least two
    synthetics configured.
02. End-to-end integration tests run against a LEZ sequencer (standalone mode)
    and are included in CI.
03. CI must be green on the default branch.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
05. A README documents end-to-end usage: deployment steps, program addresses,
    and step-by-step instructions for interacting with the program via CLI and
    mini-app.
06. Provide documentation in repo for the Logos module/library, covering the
    developer integration journey.
07. Provide documentation in repo for the CLI, covering the core user journey.
08. Provide Figma designs or equivalent for all GUI artefacts.
09. Provide a privacy and anonymisation properties document covering: what
    on-chain state and transaction data is visible to observers; what data is
    protected when the private account path is used; trust assumptions,
    specifying which guarantees are enforced by the on-chain program and which
    depend on correct client behaviour; and what happens if a user bypasses the
    expected interaction path.
10. Provide a design decision log in repo recording each major design decision
    and choice: the context, the options considered, the option chosen, and the
    rationale and trade-offs. The log is updated as decisions are made and is
    current at each milestone delivery.
11. Document testnet default values for every configuration parameter and how
    each one affects position safety.
12. For each synthetic configured on testnet, document its price feeds, their
    redundancy and their manipulation resistance.
13. For each collateral asset used in testing, document its threat model.
14. Document the cost of triggering a liquidation, including reverted attempts
    (for example when another account liquidates the position first), relative
    to the configured liquidation penalty, and the MEV considerations of the
    liquidation flow.

#### + Security

1. Provide a threat model covering at least price-feed manipulation, collateral
   price collapse, stability pool exhaustion, and misuse of the admin and freeze
   authorities.
2. The program undergoes an independent third-party security audit prior to any
   mainnet deployment, and the audit report is published. Proposals may include
   the audit in their scope and budget or leave it to be procured separately,
   and must state which.

#### + Privacy

1. Users can hold sASSET and collateral in private accounts and interact with
   the program from them.
2. Position and stability pool state are held in public accounts, so that any
   account can check a position's c-ratio and trigger its liquidation.
3. Before a user confirms an operation from a private account, the GUI shows
   what will be recorded in public state (position changes, amounts, timing).

### Soft Requirements

1. **Lending-market substrate.** Position accounting (collateral, debt,
   c-ratios, price-feed valuation) may be built on an existing LEZ lending
   market, such as one implementing
   [RFP-008](./RFP-008-lending-borrowing-protocol.md), in the style of
   MakerDAO's Direct Deposit modules, instead of standalone. The stability pool
   is still required, and the program must still work without the lending
   market.
2. **Keeper reference software.** Reference software that third parties may
   choose to run to monitor positions and trigger liquidations.
3. **Formal verification** of Reliability #1 to #3.

### Out of Scope

- Governance token design and distribution.
- Operating price feeds, keepers or any other infrastructure the program relies
  on.
- Mainnet deployment. Mainnet deployment is not part of this RFP: whether, when
  and by whom the software is deployed is a matter for those who deploy it.
- The identity of persons holding the admin and freeze authorities, and the
  exercise of those authorities, including the choice of collateral, synthetics,
  price feeds, fee recipients and parameter values, in any deployment, are
  matters for the relevant independent deployer or operator and are outside the
  scope of this RFP.

## ⚠ Platform Dependencies

The program depends on the capabilities below on LEZ testnet 0.4, with status as
at the date of this specification. Specific RFPs are named only as illustrative
examples. Proposals state which implementation of each capability they build
against, and how a change to its interface would be absorbed.

1. **Price feeds.** An ASSET/USD price feed for each demonstrated synthetic,
   sourced from outside LEZ and following the RFP-019 price data standard
   (Functionality #9). [RFP-020](./RFP-020-redstone-oracle-adaptor.md) specifies
   an example. If no such feed is available on testnet 0.4, proposals include a
   test price-feed program and say so.
2. **Fungible tokens.** A fungible token implementation for sASSET and for test
   collateral. If none is available on testnet 0.4, the proposal implements its
   own.
3. **Admin and freeze authority.** Bounded admin and freeze capabilities for
   Functionality #19 to #21. [RFP-001](./RFP-001-admin-authority-lib.md) and
   [RFP-002](./RFP-002-freeze-authority-lib.md) specify examples.

## 👤 Recommended Team Profile

Team experienced with:

- CDP and stablecoin protocol design (Liquity, MakerDAO, Synthetix or similar),
  in particular liquidation and stability pool mechanics
- Price oracle integration and manipulation resistance
- Rust and zkVM program development (RISC Zero), or Solana/SVM program
  development
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **12–16 weeks**.

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.

## Resources

- [RFP-001 — Admin Authority Library](./RFP-001-admin-authority-lib.md)
- [RFP-002 — Freeze Authority Library](./RFP-002-freeze-authority-lib.md)
- [RFP-008 — Lending & Borrowing Protocol](./RFP-008-lending-borrowing-protocol.md)
- [RFP-019 — On-Chain TWAP Oracle](./RFP-019-twap-oracle.md) (canonical oracle
  price data standard)
- [RFP-020 — RedStone Off-Chain Oracle Adaptor for LEZ](./RFP-020-redstone-oracle-adaptor.md)
- [Logos Execution Zone repository](https://github.com/logos-blockchain/logos-execution-zone)
- [LEZ actor program development book](https://github.com/logos-blockchain/logos-execution-zone/tree/moudy/guide-mdbook-fixes/docs/actor-program-development-book/src)
  (draft)
- [Liquity v1 repository](https://github.com/liquity/dev) (stability pool and
  liquidation design)
- [Synthetix SIP-302 (Pools V3)](https://sips.synthetix.io/sips/sip-302)
- [Synthetix: Hadar release synths](https://blog.synthetix.io/new-synths-update-for-the-upcoming-hadar-release/)

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
