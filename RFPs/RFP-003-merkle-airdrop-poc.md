---
id: RFP-003
title: Merkle Airdrop
tier: S
funding: $XXXXX
status: open
category: Developer Tooling & Infrastructure
---


# RFP-003 — Merkle Airdrop

## 🧭 Overview

Build a proof-of-concept that demonstrates a Merkle-based token airdrop
distributor on LEE — the equivalent of MerkleDistributor on EVM. The program
stores a Merkle root on-chain alongside a token vault, and allows any eligible
recipient to claim their allocation by submitting a valid Merkle proof.

This RFP targets a standalone PoC to validate the approach. A follow-up RFP
will be issued to package the pattern into a reusable developer library for
the Logos ecosystem.

## 🔥 Why This Matters

Token distributions are one of the most fundamental operations in any
on-chain ecosystem. Without a standardised, auditable distributor pattern,
every team building on LEE must design their own mechanism — leading to
duplicated effort, inconsistent security, and a poor experience for claimants.

A Merkle-based approach is efficient by design: only the root is stored
on-chain, while individual allocations remain off-chain until claimed. This
keeps deployment costs low regardless of the number of recipients, and makes
the full allocation set verifiable by anyone.

Delivering this primitive early accelerates the pace at which projects can launch
on LEZ, attract users, and distribute ownership. It also lays the groundwork
for LEE-native privacy features: the same architecture can optionally be
extended to hide the recipient's identity or amount on-chain.

## ✅ Scope of Work

### Hard Requirements
#### Functionality
1. An authority can initialise a distributor account storing a Merkle root,
  a funded token vault, and configuration.
2. A claimant submits a Merkle proof; the program verifies it on-chain against
  the stored root.
3. On a valid proof, tokens are transferred from the vault to the recipient
  encoded in the leaf.
4. Each leaf can only be claimed once; double-claiming is prevented using a
  per-claim record account.
#### Supportability
1. Demonstrated as a sample app; the choice of token and amounts does not
  matter — the focus is on the Merkle proof and claim pattern.

### Soft Requirements
If possible.
#### Functionality
5. Leaf encoding includes both the recipient address and the claimable amount;
  the program validates both against the submitted proof.
#### +Privacy (bonus)
1. An optional privacy-circuit variant verifies Merkle inclusion while keeping
  the leaf (recipient and amount) private, emitting only a public nullifier
  to prevent double-claims — in line with LEE's nullifier-uniqueness rule.

### Out-of-scope

Vesting schedules (linear unlock, cliff vesting) are not in scope for this PoC
and will be addressed in a follow-up RFP.

## 👤 Recommended Team Profile

Developer experienced with:

- Solana or SVM program development (Anchor or native)
- Merkle tree construction and on-chain proof verification
- Token program interactions and PDA-based vault authority
- Writing and running on-chain tests (e.g. Bankrun, Anchor tests)

## ⏱ Timeline Expectations

Estimated duration: **3 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

TODO: LEE official doc

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
