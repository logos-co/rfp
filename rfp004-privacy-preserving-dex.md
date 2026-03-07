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

Build a decentralized exchange that enables users to trade shielded assets
within the Logos ecosystem without revealing their identities, order sizes,
or trading strategies. The DEX must operate entirely within the shielded
environment, providing MEV-resistant, fair trading for all participants.

The DEX is the most critical in-ecosystem application. It allows users to
trade their newly bridged and shielded assets anonymously, preventing the
de-anonymization that would occur if they had to withdraw to a centralized
exchange. Without a private DEX, the ecosystem cannot sustain meaningful
economic activity.

The team building this should have deep experience in AMM or order-book
design, ZK circuit development, and MEV-resistant trading mechanisms such
as frequent batch auctions or encrypted mempools.

## 🔥 Why This Matters

On transparent chains, DEXs consistently emerge as the dominant application
category. On Solana, Raydium's TVL surpassed $1B, representing a third of
the chain's entire DeFi TVL. On Base, 12 of the 23 initial dApps with over
$1M TVL were DEXs. Trading is the primary initial activity on any new chain.

On transparent chains, this trading activity comes with severe downsides:
front-running and sandwich attacks are rampant, extracting billions in MEV
from ordinary users. A private DEX on Logos eliminates these attack vectors
by design — trades are submitted in encrypted form, and mechanisms like
frequent batch auctions ensure that no party can observe and exploit pending
orders. This is not just a privacy improvement; it is a fundamentally fairer
trading environment that serves as a key differentiator for the Logos
ecosystem.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Implement an automated market maker (AMM) that operates on shielded
   assets within the Logos ecosystem.
2. Support creation of liquidity pools for arbitrary shielded token pairs.
3. Liquidity providers must be able to add and remove liquidity without
   revealing their positions or addresses publicly.
4. Traders must be able to swap shielded assets without revealing trade
   size, direction, or identity.
5. Implement a MEV-resistant trade execution mechanism (e.g., frequent
   batch auctions, commit-reveal schemes, or encrypted mempool
   integration).
6. Support configurable fee tiers for liquidity pools.

#### Security
1. All trade execution logic must be verifiable via ZK proofs — the
   protocol must prove correct execution without revealing trade details.
2. Implement slippage protection and minimum output guarantees for
   traders.
3. Include an emergency pause mechanism for pools in the event of a
   detected exploit.

#### Usability
1. Provide a minimal front-end UI for swapping, pool creation, and
   liquidity management.
2. Provide a pool analytics dashboard showing aggregate volume and
   TVL (without revealing individual positions).

### Soft Requirements

1. Support for concentrated liquidity positions (similar to Uniswap V3)
   within the shielded environment.
2. Support for limit orders executed within the batch auction framework.
3. Router functionality for multi-hop swaps across pools.
4. Integration with the Logos bridge (RFP-003) for seamless bridge-and-swap
   user flows.
5. Exploration of order-book-based designs as an alternative or complement
   to AMM pools.

## 👤 Recommended Team Profile

Team experienced with:

- AMM and/or order-book DEX design and implementation
- Zero-knowledge proof systems and circuit development
- MEV research and mitigation techniques
- DeFi smart contract development and security
- Front-end development for trading applications
- Shielded transaction architectures

## ⏱ Timeline Expectations

Estimated duration: **14–22 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [Logos Blockchain Node Quickstart](https://github.com/logos-co/logos-docs/blob/main/docs/blockchain/quickstart-guide-for-the-logos-blockchain-node.md)
- TODO: LEE official doc
- TODO: Shielded asset specification


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
