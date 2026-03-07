---
id: RFP-008
title: Private NFT Marketplace
tier: L
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-008 — Private NFT Marketplace

## 🧭 Overview

Build a marketplace for non-fungible tokens (NFTs) where ownership, bids,
transaction history, and collector activity are confidential by default.
The marketplace must operate within the Logos shielded ecosystem, enabling
private minting, trading, and transfer of unique digital assets.

NFTs serve as the cultural on-ramp for blockchain ecosystems, attracting
users beyond the DeFi-native audience. A private NFT marketplace unlocks
use cases that are impossible on transparent chains: anonymous credentials,
private digital identity, confidential collectibles, and sovereign digital
ownership where holding a high-value asset does not make you a target.

The ideal team has experience in NFT marketplace design, shielded token
standards for non-fungible assets, and front-end development for
discovery-driven applications.

## 🔥 Why This Matters

NFT marketplaces drive massive user engagement and transaction volume.
Magic Eden processed over $6B in volume and at times captured over 90% of
Solana's NFT market. Base's "Onchain Summer" campaign used daily NFT mints
to drive millions of transactions and onboard a wave of new users who were
not primarily DeFi-focused.

On transparent chains, NFT ownership is fully public. This means anyone can
see exactly which wallets hold valuable assets, track collector behavior,
and target high-value holders for social engineering or physical threats.
Bids and offers are visible, enabling strategic manipulation. A private
NFT marketplace on Logos solves these problems: ownership is shielded,
bids are confidential, and transfers leave no public trace. This protects
collectors, enables truly sovereign digital identities, and opens the door
for privacy-sensitive use cases like anonymous credentials and private
access tokens.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Support minting of NFTs directly into the Logos shielded pool — the
   creator's identity and the token's initial ownership are private.
2. Support listing NFTs for sale with shielded pricing (the listing
   price is visible to potential buyers but the seller's identity is
   not).
3. Support private bidding on listed NFTs — bid amounts and bidder
   identities are confidential until the seller accepts.
4. Support direct private transfers of NFTs between shielded addresses.
5. Support royalty enforcement on secondary sales, with royalty
   payments flowing to the creator's shielded address.

#### Security
1. Implement provenance verification via ZK proofs — buyers can verify
   that an NFT was minted by a specific creator (or collection) without
   revealing the full ownership history.
2. Include an emergency pause mechanism for marketplace operations.
3. Implement escrow mechanisms for safe trade execution.

#### Usability
1. Provide a front-end UI for browsing collections, minting, listing,
   bidding, and purchasing NFTs.
2. Support collection-level pages with aggregate statistics (floor
   price, volume) without revealing individual holder information.
3. Provide creator tools for launching collections with configurable
   minting parameters.

### Soft Requirements

1. Support for auction mechanisms (English auctions, Dutch auctions)
   with private bidding.
2. Support for NFT bundles and batch transfers.
3. Integration with the Logos DEX (RFP-004) for NFT-fi use cases
   (fractionalization, NFT-collateralized lending).
4. Support for metadata storage via Logos Storage module for
   decentralized, persistent NFT content.
5. Exploration of private identity and credential NFTs — non-transferable
   tokens that attest to properties without revealing them (e.g.,
   "holder is over 18" without revealing age or identity).

## 👤 Recommended Team Profile

Team experienced with:

- NFT marketplace design and implementation
- Non-fungible token standards and metadata handling
- Zero-knowledge proof systems
- Smart contract development and security
- Front-end development for consumer-facing applications
- IPFS / decentralized storage integration

## ⏱ Timeline Expectations

Estimated duration: **12–18 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [Logos Storage Module](https://logos-storage-docs.netlify.app/tutorials/storage-module/)
- [Logos Blockchain Node Quickstart](https://github.com/logos-co/logos-docs/blob/main/docs/blockchain/quickstart-guide-for-the-logos-blockchain-node.md)
- TODO: LEE official doc
- TODO: Shielded NFT token standard specification


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
