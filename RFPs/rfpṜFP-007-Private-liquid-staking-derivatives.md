---
id: RFP-007
title: Private Liquid Staking Derivatives (LSDs)
tier: L
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-007 — Private Liquid Staking Derivatives (LSDs)

## 🧭 Overview

Build a liquid staking protocol for the Logos network that allows users to
stake the native token to secure the network while receiving a liquid,
shielded derivative token that can be used freely across the private DeFi
ecosystem. The entire flow — staking, receiving the liquid token, and using
it — must remain within the shielded environment.

Liquid staking is the network aligner: it solves the tension between
securing the chain (which requires locking capital) and maintaining
liquidity (which requires freely available capital). Without liquid staking,
users must choose between earning staking yield and participating in DeFi,
fragmenting the ecosystem's capital efficiency.

The ideal team has experience in PoS staking infrastructure, liquid staking
protocol design, and privacy-preserving token issuance.

## 🔥 Why This Matters

Liquid staking has proven to be one of the most demanded primitives on PoS
chains. Marinade Finance became Solana's largest protocol with over $1.8B
in TVL, demonstrating immense user demand to earn staking yield without
sacrificing liquidity. On Ethereum, liquid staking (Lido, Rocket Pool)
represents the single largest DeFi category by TVL.

For a privacy L1, liquid staking carries additional privacy implications.
On transparent chains, staking is inherently public — the staker's address,
amount staked, and chosen validator are all visible. This creates a direct
link between a user's staking activity (often tied to KYC'd validators)
and their broader financial activity. A private liquid staking protocol
on Logos must sever this link: the act of staking, the receipt of the
liquid derivative token, and the subsequent use of that token in DeFi must
all be shielded, ensuring that validator-level information cannot be used
to de-anonymize users.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Users can stake the native Logos token through the protocol and
   receive a shielded liquid staking derivative (e.g., sLOGOS) in
   return.
2. The staking delegation to validators must not reveal which end-user
   initiated the stake — the protocol acts as an intermediary pool.
3. Staking rewards must accrue to the liquid derivative token
   (rebasing or exchange-rate model) without revealing individual
   staker rewards.
4. Users can unstake by returning the derivative token, subject to
   the network's unbonding period, with the redemption happening
   privately.
5. The liquid derivative token must be a standard shielded token
   compatible with other Logos DeFi protocols (DEX, lending, etc.).

#### Security
1. Implement validator set diversification to avoid concentration
   risk — the protocol should distribute stake across multiple
   validators.
2. Include slashing protection mechanisms — if a validator is slashed,
   the loss is socialized across the pool rather than hitting
   individual stakers.
3. Include an emergency pause mechanism for staking and unstaking
   operations.

#### Usability
1. Provide a minimal front-end UI for staking, unstaking, and
   viewing aggregate protocol statistics.
2. Display aggregate staking metrics (total staked, current APY,
   validator distribution) without revealing individual positions.

### Soft Requirements

1. Governance mechanism for validator set curation and protocol
   parameter adjustments.
2. Integration with the Logos DEX (RFP-004) for liquid derivative
   token trading.
3. Integration with the lending protocol (RFP-006) to use the
   liquid derivative as collateral.
4. Support for instant unstaking via a liquidity buffer pool.
5. Exploration of MEV-sharing mechanisms where staking rewards include
   a portion of MEV revenue.

## 👤 Recommended Team Profile

Team experienced with:

- PoS staking infrastructure and validator operations
- Liquid staking protocol design (Lido, Marinade, or similar)
- Zero-knowledge proof systems
- DeFi smart contract development and security
- Token economic modeling
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **12–18 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [Logos Blockchain Node Quickstart](https://github.com/logos-co/logos-docs/blob/main/docs/blockchain/quickstart-guide-for-the-logos-blockchain-node.md)
- TODO: LEE official doc
- TODO: Logos staking and consensus specification
- TODO: Shielded asset specification


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
