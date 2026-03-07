---
id: RFP-011
title: Private Social & Web3 Social Application
tier: L
funding: $XXXXX
status: open
category: Applications & Integrations
---


# RFP-011 — Private Social & Web3 Social Application

## 🧭 Overview

Build a social application on the Logos stack where users' social graphs,
content, interactions, and financial activity are private by default. The
application should leverage Logos Messaging for peer-to-peer communication
and the shielded ecosystem for any on-chain social-economic interactions,
providing a viral, user-friendly experience that can onboard a massive wave
of non-DeFi-native users.

Social applications are the network effect multiplier for any blockchain
ecosystem. They attract users who are not primarily motivated by trading
or yield but by community, identity, and communication. A privacy-preserving
social app is uniquely compelling: it protects social connections from
public surveillance, prevents social-graph-based targeting, and enables
authentic interactions without the panopticon effect of transparent chains.

The ideal team has experience in social application design, peer-to-peer
messaging protocols, and building viral consumer products with strong
privacy guarantees.

## 🔥 Why This Matters

Social applications can drive explosive user growth. Friend.tech on Base
became a sensation overnight, generating over $1M in daily fees and driving
Base's daily transactions to new highs. It demonstrated that a social
mechanic tied to on-chain economics can onboard users at a scale and speed
that no DeFi protocol can match.

However, Friend.tech also exposed the fundamental flaw of social apps on
transparent chains: every user's social connections, transaction history,
and financial interactions were publicly visible. Users' real-world
identities could be linked to their on-chain social activity, and the
economic dynamics of their relationships were exposed for all to see.
A private social application on Logos solves this entirely — social graphs
are shielded, interactions happen over Logos Messaging (with mix-net
privacy), and any on-chain economic activity (tipping, group payments,
social tokens) happens within the shielded pool. This enables a social
experience that is simultaneously on-chain and private, unlocking use cases
that are impossible on transparent chains: anonymous communities, private
group coordination, and confidential social-economic interactions.

## ✅ Scope of Work

### Hard Requirements

#### Functionality
1. Users can create pseudonymous profiles tied to their shielded Logos
   identity — no public link to any external identity or address.
2. Users can form social connections (follow, friend, group membership)
   without these connections being publicly visible on-chain.
3. Support private messaging between users, leveraging the Logos
   Messaging (Delivery Module or Chat Module) for peer-to-peer
   encrypted communication.
4. Support on-chain social-economic interactions within the shielded
   pool — e.g., tipping, social token issuance, paid group access,
   or content gating.
5. Implement a discovery mechanism that allows users to find and
   connect with others without revealing their full social graph.

#### Security
1. Social graph data must not be stored or derivable from on-chain
   state — connections should be maintained off-chain or via encrypted
   on-chain state.
2. All economic interactions (tips, payments, social token trades) must
   happen within the Logos shielded pool.
3. Include content moderation mechanisms that are compatible with
   privacy (e.g., community-based moderation, user-side filtering)
   without centralized surveillance.

#### Usability
1. Provide a mobile-first or responsive web front-end with a consumer
   social app experience (feed, profiles, messaging, notifications).
2. Onboarding flow must be simple enough for non-crypto-native users —
   abstract wallet management and key handling behind familiar UX
   patterns.
3. Support push notifications for social interactions without leaking
   metadata about the sender or content.

### Soft Requirements

1. Support for social tokens or "keys" that grant access to exclusive
   content or groups, traded within the shielded DEX (RFP-004).
2. Integration with the NFT marketplace (RFP-008) for profile pictures,
   badges, and social collectibles.
3. Support for decentralized content feeds with privacy-preserving
   content recommendations.
4. Integration with the Logos AnonComms mix-net for metadata-private
   messaging.
5. Exploration of reputation systems built on ZK proofs — users can
   prove reputation metrics (e.g., "active for 6 months", "trusted
   by N connections") without revealing their identity or full
   activity history.
6. Support for anonymous group coordination tools (voting, scheduling,
   shared treasuries).

## 👤 Recommended Team Profile

Team experienced with:

- Social application design and viral growth mechanics
- Peer-to-peer messaging protocols and encrypted communication
- Consumer mobile/web application development
- Zero-knowledge proof systems
- Privacy-preserving identity and reputation systems
- UX design for non-crypto-native users

## ⏱ Timeline Expectations

Estimated duration: **14–22 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [Logos Documentation](https://github.com/logos-co/logos-docs)
- [Logos Messaging — Delivery Module API](https://github.com/logos-co/logos-docs/blob/main/docs/messaging/journeys/use-the-logos-delivery-module-api-from-an-app.md)
- [Logos Messaging — Chat Module API](https://github.com/logos-co/logos-docs/blob/main/docs/messaging/journeys/use-the-logos-chat-module-api-from-an-app.md)
- [Logos AnonComms Mixnet](https://github.com/logos-co/logos-docs/blob/main/docs/connect/anoncomms/journeys/discover-nodes-and-send-messages-via-the-anoncomms-mixnet-demo-app.md)
- [Logos Blockchain Node Quickstart](https://github.com/logos-co/logos-docs/blob/main/docs/blockchain/quickstart-guide-for-the-logos-blockchain-node.md)
- TODO: LEE official doc


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
