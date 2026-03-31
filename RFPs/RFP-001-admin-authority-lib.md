---
id: RFP-001
title: Admin Authority Library
tier: XS
funding: $XXXXX
status: open
category: Developer Tooling & Infrastructure
---


# RFP-001 — Admin Authority Library

## 🧭 Overview

Build a reusable library that provides standardised access control for LEE
programs, where privileged functions, including the ability to transfer or
renounce authority, can only be called by an admin authority.

The library must be importable as a dependency by any LEE program and ship
with documentation and usage examples so teams can integrate the pattern
without re-implementing it from scratch.

## 🔥 Why This Matters

As the Logos ecosystem grows, programs deployed on LEZ need foundational
security primitives from day one. Without standardised access control,
every team must design their own — leading to inconsistent implementations,
duplicated effort, and a higher risk of critical vulnerabilities.

Delivering this as a shared library lowers the barrier for developers building
on LEE. Teams can focus on application logic rather than re-inventing admin
authority patterns, accelerating the pace at which new programs ship.

## ✅ Scope of Work

### Hard Requirements
#### Functionality
1. Admin authority is set at program initialisation.
2. Admin authority can transfer admin authority to a new signer.
3. Admin authority can revoke admin authority, effectively renouncing
  admin control.
4. Admin authority is the only one that can call privileged instructions
  exposed by the library (demonstrated via a gated `config` PDA update).
#### Usability
1. There can only be one admin authority (signer) at a time.
2. The library is integrated into the [SPEL framework](https://github.com/logos-co/spel)
  so that programs using SPEL can enable admin authority with minimal
  boilerplate — ideally a single annotation or configuration flag.
3. Documentation includes at least one end-to-end usage example showing
  how a SPEL program gates its own instructions behind the admin authority.
#### Supportability
1. Unit and integration tests cover all hard requirements and run in CI.
2. A sample program that imports the library is included to validate the
  integration path and serve as a reference for consumers.
### Soft Requirements
If possible.
#### Reliability
1. Admin authority can only be set to a valid new signer (on-curve key
  or deployed PDA), when set or initialised.

## 👤 Recommended Team Profile

Developer experienced with:

- Solana or SVM program development (Anchor or native)
- Access control and authority patterns in on-chain programs
- PDA derivation and account validation
- Writing and running on-chain tests (e.g. Bankrun, Anchor tests)
- Library/crate packaging and documentation

## ⏱ Timeline Expectations

Estimated duration: **4 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

- [SPEL framework](https://github.com/logos-co/spel)
- TODO: LEE official doc

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
