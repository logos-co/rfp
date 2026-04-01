---
id: RFP-009
title: Private Derivatives & Perpetual Trading Platform
tier: XL
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-009 — Private Derivatives & Perpetual Trading Platform

## 🧭 Overview

Build a derivatives and perpetual futures trading platform that operates
entirely within the Logos shielded ecosystem, allowing traders to open
leveraged positions on shielded assets with full confidentiality over
their collateral, position sizes, entry prices, and liquidation levels.

Derivatives trading is the sophisticated trading venue of any mature
blockchain ecosystem. It attracts professional traders, generates
substantial fee revenue, and dramatically increases trading volume beyond
what spot markets alone can achieve. On a privacy chain, confidential
derivatives are a powerful differentiator — traders can execute strategies
without revealing their positioning to competitors or predatory MEV bots.

The ideal team has deep experience in perpetual futures mechanism design,
funding rate models, ZK circuit development for complex financial logic,
and trading platform UI/UX.

## 🔥 Why This Matters

Derivatives platforms are among the highest revenue-generating applications
in crypto. On Arbitrum, GMX generated more fee revenue than nearly any
other dApp globally, demonstrating the enormous economic potential of
on-chain derivatives. Berachain launched with a native perpetuals DEX
(Berps) from day one to capture this market immediately.

On transparent chains, derivatives trading creates extreme privacy
vulnerabilities: every trader's position size, leverage, collateral, and
liquidation price are publicly visible. This enables targeted liquidation
hunting, where well-capitalized actors deliberately move prices to trigger
visible liquidation levels. It also exposes trading strategies to
competitors. A private derivatives platform on Logos protects all of these
details — collateral, positions, and liquidations are all confidential.
This creates a fundamentally fairer trading environment and represents a
significant ecosystem value proposition for attracting professional
trading activity.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Support perpetual futures contracts on shielded asset pairs with
   configurable leverage (e.g., up to 50x).
2. Traders can open, modify, and close positions entirely within the
   shielded environment — position size, collateral, leverage, and
   PnL are all confidential.
3. Implement a funding rate mechanism that balances long and short
   interest, calculated on aggregate open interest without revealing
   individual positions.
4. Implement a private liquidation mechanism — positions that fall
   below maintenance margin are liquidatable, but the specific
   position and trader are not publicly identifiable.
5. Implement a price oracle integration for mark price and index
   price calculations with manipulation-resistant design.
6. Support both market orders and limit orders with private execution.

#### Security
1. All position management operations must be verifiable via ZK proofs,
   ensuring correct execution (leverage limits, margin requirements,
   liquidation thresholds) without revealing individual details.
2. Implement an insurance fund mechanism to cover losses from
   under-collateralized liquidations, funded by a portion of trading
   fees.
3. Include an emergency pause mechanism to halt trading in the event
   of oracle failure or detected exploit.
4. Implement position size limits and open interest caps during the
   initial launch phase.

#### Usability
1. Provide a trading front-end with real-time price charts, order
   entry, and position management.
2. Provide aggregate market statistics (total open interest, 24h
   volume, funding rates) without revealing individual positions.

### Soft Requirements

1. Support for additional derivative types beyond perpetuals (e.g.,
   options, structured products).
2. Integration with the Logos DEX (RFP-004) for spot-perps arbitrage
   pathways.
3. Integration with the lending protocol (RFP-006) for cross-margining.
4. Support for portfolio margining across multiple positions.
5. Keeper network for decentralized liquidation execution.
6. Exploration of an order-book model as an alternative to the
   vault/pool model.

## 👤 Recommended Team Profile

Team experienced with:

- Perpetual futures and derivatives protocol design (GMX, dYdX, or similar)
- Zero-knowledge proof systems and advanced ZK circuit development
- Funding rate and liquidation mechanism design
- Oracle integration and price feed security
- High-performance trading system architecture
- Front-end development for trading platforms

## ⏱ Timeline Expectations

Estimated duration: **18–26 weeks**


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
