---
id: RFP-006
title: Private Lending & Borrowing Protocol
tier: XL
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-006 — Private Lending & Borrowing Protocol

## 🧭 Overview

Build a lending and borrowing protocol that operates entirely within the
Logos shielded ecosystem, allowing users to collateralize their shielded
assets to borrow other shielded assets — all without publicly revealing
their financial positions, debt levels, or net worth.

Lending is the capital efficiency engine of any DeFi ecosystem. It unlocks
idle capital by allowing holders to earn yield on deposits and borrowers to
access liquidity without selling their assets. On a privacy chain, this
means users can engage in sophisticated financial strategies without
exposing themselves to targeted attacks based on visible on-chain positions.

This is one of the most technically challenging applications in the privacy
DeFi stack, requiring advanced ZK circuits for private liquidations and
interest rate calculations. The team should have deep expertise in DeFi
protocol design, ZK proof systems, and liquidation mechanics.

## 🔥 Why This Matters

Lending protocols are proven ecosystem catalysts. Aave's deployment on
Polygon, incentivized with $40M in rewards, led to over $1B in TVL and
catalyzed the entire Polygon DeFi ecosystem. Lending protocols lock
significant capital, create demand for stablecoins, and generate
composability opportunities across the ecosystem.

On transparent chains, however, lending creates serious privacy risks:
anyone can see a user's collateral, debt, and liquidation price. This
exposes users to targeted liquidation attacks, social engineering based on
visible wealth, and strategic manipulation of their positions. A private
lending protocol on Logos eliminates these risks entirely — positions,
collateral ratios, and liquidations all happen within the shielded
environment. This is a key differentiator that enables a truly private
financial architecture and protects users from being targeted based on
their financial activity.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Users can deposit shielded assets as collateral without revealing
   the deposit amount or their identity.
2. Users can borrow shielded assets against their collateral without
   revealing the borrow amount, collateral ratio, or identity.
3. Implement an interest rate model (e.g., utilization-based) that
   adjusts rates based on aggregate pool utilization without revealing
   individual positions.
4. Implement a private liquidation mechanism — positions that fall
   below the required collateral ratio must be liquidatable, but the
   specific position being liquidated must not be publicly identifiable
   before or during liquidation.
5. Users can repay borrowed assets and withdraw collateral privately.

#### Security
1. All lending operations (deposit, borrow, repay, withdraw, liquidate)
   must be verifiable via ZK proofs, ensuring correct execution without
   revealing individual details.
2. Implement oracle integration for collateral price feeds with
   manipulation-resistant design.
3. Include an emergency pause mechanism to halt lending operations
   in the event of a detected exploit or oracle failure.
4. Implement bad debt socialization or insurance fund mechanisms to
   protect depositors.

#### Usability
1. Provide a minimal front-end UI for depositing, borrowing, repaying,
   and managing positions.
2. Provide an aggregate protocol health dashboard (total deposits,
   total borrows, utilization rates) without revealing individual
   positions.

### Soft Requirements

1. Support for multiple collateral and borrow asset types.
2. Integration with the Logos DEX (RFP-004) for liquidation execution.
3. Integration with the private stablecoin (RFP-005) as a primary
   borrow asset.
4. Flash loan support within the shielded environment.
5. Governance mechanisms for adjusting protocol parameters (interest
   rate curves, collateral factors, liquidation penalties).
6. Exploration of isolated lending markets for higher-risk assets.

## 👤 Recommended Team Profile

Team experienced with:

- DeFi lending protocol design (Aave, Compound, or similar)
- Zero-knowledge proof systems and advanced ZK circuit development
- Liquidation mechanism design and MEV considerations
- Oracle integration and price feed security
- Smart contract security and formal verification
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **16–24 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [Logos Blockchain Node Quickstart](https://github.com/logos-co/logos-docs/blob/main/docs/blockchain/quickstart-guide-for-the-logos-blockchain-node.md)
- TODO: LEE official doc
- TODO: Shielded asset specification
- TODO: Oracle integration guide


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
