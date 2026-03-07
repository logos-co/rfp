---
id: RFP-003
title: Privacy-Preserving Bridge & Cross-Chain Infrastructure
tier: XL
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-003 — Privacy-Preserving Bridge & Cross-Chain Infrastructure

## 🧭 Overview

Build a production-grade, privacy-preserving bridge that enables users to
onboard assets from transparent chains (e.g., Ethereum, L2s) into the Logos
shielded ecosystem without creating a public link between their source and
destination addresses.

Without a secure bridge, the Logos L1 remains an isolated network with no
value to shield. This is the single most critical piece of infrastructure
for bootstrapping the ecosystem — it is the lifeline through which all
initial capital and users flow.

The bridge must be designed as a one-way deposit mechanism into the shielded
pool, breaking the on-chain link between a user's transparent-chain identity
and their new private address on Logos. The team building this should have
deep experience in cross-chain messaging, ZK proof systems, and bridge
security.

## 🔥 Why This Matters

History shows that bridges are the primary catalyst for ecosystem growth.
Base onboarded over $200M in its first weeks via bridges, and Polygon's PoS
bridge held approximately $1.3B in stablecoins alone. This demonstrates that
massive value is ready to flow into new ecosystems the moment a trusted
pathway exists.

For a privacy-focused L1, the stakes are even higher. A standard bridge
would link a user's transparent wallet (e.g., on Ethereum) to their new
address on Logos, immediately defeating the purpose of the chain's privacy
guarantees. The bridge *must* sever this link by design — depositing assets
directly into the shielded pool so that incoming users are private from the
very first transaction. Without this, every subsequent privacy application
(DEX, lending, stablecoins) is undermined at the root.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Support bridging of major assets (ETH, USDC, USDT, WBTC at minimum)
   from Ethereum mainnet into the Logos shielded pool.
2. Deposits from the source chain must enter the Logos shielded pool
   directly — there must be no publicly visible mapping between the
   source address and the recipient's shielded address.
3. Implement a secure lock-and-mint or burn-and-mint mechanism with
   on-chain proof verification.
4. Provide a withdrawal path from the Logos shielded pool back to
   transparent chains, with configurable anonymity set considerations
   (e.g., time delays, batched withdrawals).
5. Implement a relayer network or mechanism so that users do not need
   to hold native Logos tokens to complete their first bridge transaction.

#### Security
1. Bridge contracts must be auditable and designed to minimize trust
   assumptions (e.g., light-client verification, ZK proof of state).
2. Include an emergency pause mechanism (freeze authority) that can halt
   the bridge in the event of a detected exploit.
3. Implement rate limiting and deposit caps during the initial launch
   phase.

#### Usability
1. Provide a minimal front-end UI for bridge operations (deposit and
   withdrawal).
2. Provide clear transaction status tracking from source chain to
   shielded deposit confirmation.

### Soft Requirements

1. Support for additional source chains beyond Ethereum (e.g., Arbitrum,
   Optimism, Solana).
2. Integration with existing cross-chain messaging protocols (e.g.,
   IBC, LayerZero) where applicable.
3. Support for batched deposits to improve the anonymity set of incoming
   users.
4. Exploration of trustless bridge designs using ZK light clients for
   source-chain state verification.

## 👤 Recommended Team Profile

Team experienced with:

- Cross-chain bridge architecture and security
- Zero-knowledge proof systems (SNARKs/STARKs)
- Smart contract development on EVM and SVM-based chains
- Shielded pool and commitment scheme design
- Production bridge deployments and security auditing
- Front-end development for DeFi applications

## ⏱ Timeline Expectations

Estimated duration: **12–20 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [Logos Blockchain Node Quickstart](https://github.com/logos-co/logos-docs/blob/main/docs/blockchain/quickstart-guide-for-the-logos-blockchain-node.md)
- TODO: LEE official doc
- TODO: Shielded pool specification


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
