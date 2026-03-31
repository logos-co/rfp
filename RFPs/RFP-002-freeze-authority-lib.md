---
id: RFP-002
title: Freeze Authority Library
tier: XS
funding: $XXXXX
status: open
category: Developer Tooling & Infrastructure
---


# RFP-002 — Freeze Authority Library

## 🧭 Overview

Build a reusable library that provides a standardised freeze mechanism for
LEE programs, allowing an authorised account to disable all (or selected)
interactions as a circuit breaker when an issue is discovered.

The library must be importable as a dependency by any LEE program and ship
with documentation and usage examples so teams can integrate the pattern
without re-implementing it from scratch.

## 🔥 Why This Matters

As the Logos ecosystem grows, programs deployed on LEZ need foundational
security primitives from day one. Without a standardised emergency stop
mechanism, every team must design their own — leading to inconsistent
implementations, duplicated effort, and a higher risk of critical
vulnerabilities.

Delivering this as a shared library lowers the barrier for developers
building on LEE. Teams can focus on application logic rather than
re-inventing freeze patterns, accelerating the pace at which new programs
ship.

As more programs are deployed and begin handling real value, the ability
for an authority to freeze a compromised program is the difference between
a contained incident and a catastrophic loss.

## ✅ Scope of Work

### Hard Requirements
#### Functionality
1. Freeze authority can be set at program initialisation.
2. Freeze authority can be changed by admin authority.
3. Freeze authority can freeze the program, rejecting any attempt to interact
  with it; apart from unfreezing or changing the freeze authority.
4. Freeze authority can un-freeze the program, re-enabling interactions.
5. Freeze authority can be revoked by admin authority.
#### Usability
1. There can only be one freeze authority (signer) at a time.
2. The library is integrated into the [SPEL framework](https://github.com/logos-co/spel)
  so that programs using SPEL can enable freeze authority with minimal
  boilerplate — ideally a single annotation or configuration flag.
3. Documentation includes at least one end-to-end usage example showing
  how a SPEL program integrates the freeze check.
#### Supportability
1. Unit and integration tests cover all hard requirements and run in CI.
2. A sample program that imports the library is included to validate the
  integration path and serve as a reference for consumers.

### Soft Requirements
If possible.
#### Reliability
1. Freeze authority can only be set to a valid new signer (on-curve key
  or deployed PDA), when set or initialised.

### Out-of-scope

Note that due to the privacy properties of LEE, it is not possible to freeze
actions for a given signer/account. Hence, such a feature is not expected.
Freezing is expected to be agnostic to the signers or initiators of a
transaction.

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
