---
id: RFP-000
title: Project Title
tier: XS/S/M/L/XL
funding: $XXXXX
status: open/closed
category: Developer Tooling & Infrastructure / Applications & Integrations / Ecosystem & Community Enablement
---


# RFP-001 — Program Authority - Access Control & Freeze

## 🧭 Overview

Build a proof-of-concept that demonstrates access control and freeze
authority on a LEE program, inspired by OpenZeppelin's
[Ownable](https://docs.openzeppelin.com/contracts/5.x/access-control)
and
[Pausable](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Pausable.sol)
patterns in Solidity.

The PoC should demonstrate:

- **Admin authority**: A designated account with exclusive access
  to privileged functions, including the ability to transfer or
  renounce admin authority.
- **Freeze authority**: The ability to *freeze* a program, disabling
  any (or selected) interactions. This acts as a circuit
  breaker — an emergency stop mechanism that an authorized account
  can trigger when an issue is discovered.

This is particularly useful during early-stage program development
and testing, where the ability to halt interactions quickly can
prevent exploits from causing lasting damage.

This RFP targets a standalone PoC to validate the approach. A
follow-up RFP will be issued to package the patterns into a reusable
developer library for the Logos ecosystem.

## 🔥 Why This Matters

As the Logos ecosystem grows, programs deployed on LEZ need
foundational security primitives from day one. Without standardized
access control and emergency stop mechanisms, every team must design
their own — leading to inconsistent implementations, duplicated
effort, and a higher risk of critical vulnerabilities.

Establishing these patterns early lowers the barrier for developers
building on LEE. Teams can focus on application logic rather than
re-inventing admin and freeze authority patterns, accelerating the pace at
which new programs ship. A proven PoC also gives the ecosystem a
reference implementation that builds confidence among developers and
users alike — demonstrating that LEE programs can be managed and
secured with the same rigor expected in mature smart contract
environments.

This is a prerequisite for safe experimentation: as more programs
are deployed and begin handling real value, the ability for an
authority to freeze a compromised program is the difference between
a contained incident and a catastrophic loss.

## ✅ Scope of Work

#### Hard Requirements

**Functionality**

- F1. Admin authority is set at program deployment.
- F2. Admin authority can transfer admin authority to a new
  signer.
- F3. Admin authority can transfer admin authority to the default
  program, effectively renouncing admin control.
- F4. Admin authority can set and transfer freeze authority.
- F5. Admin authority is the only one that can set a specific
  value on the PDA used by the program (sample value to
  demonstrate gated access).
- F6. Freeze authority can freeze the program, rejecting any
  attempt to interact with it; apart from admin-related
  interactions.
- F7. Freeze authority can un-freeze the program, re-enabling
  interactions.

**Usability**

- U1. There can only be one admin authority (signer) at a time.
- U2. There can only be one freeze authority (signer) at a time.

**Reliability**

- R1. Admin authority transfer can only be done to a valid new
  signer (on-curve key or deployed PDA) or to the default
  program, to prevent accidental loss of control.
- R2. Freeze authority transfer can only be done to a valid new
    signer (on-curve key or deployed PDA) or to the default
    program, to prevent accidental loss of control.

**Supportability**

- S1. Demonstrated as a sample app; the program's core
  functionality does not matter — the focus is on the authority
  patterns.

#### Soft Requirements

None.

## 👤 Recommended Team Profile

Developer experienced with:

- Solana or SVM program development (Anchor or native)
- Access control and authority patterns in on-chain programs
- PDA derivation and account validation
- Writing and running on-chain tests (e.g. Bankrun, Anchor tests)

## ⏱ Timeline Expectations

Estimated duration: **4–6 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

TODO: LEE official doc

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
