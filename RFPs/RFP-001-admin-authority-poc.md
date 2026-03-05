---
id: RFP-001
title: Admin Authority
tier: XS
funding: $XXXXX
status: open
category: Developer Tooling & Infrastructure
---


# RFP-001 — Admin Authority

## 🧭 Overview

Build a proof-of-concept that demonstrates access control on an LEE program,
where privileged functions, including the ability to transfer or renounce
authority, can only be called by an admin authority.

This is particularly useful during early-stage program development and testing,
where the ability to change specific parameters can help iterate on the program,
and potentially prevent exploits from causing lasting damage.

This RFP targets a standalone PoC to validate the approach. A follow-up RFP
will be issued to package the pattern into a reusable developer library for
the Logos ecosystem.

## 🔥 Why This Matters

As the Logos ecosystem grows, programs deployed on LEZ need foundational
security primitives from day one. Without standardised access control,
every team must design their own — leading to inconsistent implementations,
duplicated effort, and a higher risk of critical vulnerabilities.

Establishing these patterns early lowers the barrier for developers building
on LEE. Teams can focus on application logic rather than re-inventing admin
authority patterns, accelerating the pace at which new programs ship. A proven
PoC also gives the ecosystem a reference implementation that builds confidence
among developers and users alike — demonstrating that LEE programs can be
managed and secured with the same rigour expected in mature smart contract
environments.

## ✅ Scope of Work

### Hard Requirements
#### Functionality
1. Admin authority is set at program initialisation.
2. Admin authority can transfer admin authority to a new signer.
3. Admin authority can revoke admin authority, effectively renouncing
  admin control.
4. Admin authority is the only one that can set a specific value on the
  unique `config` PDA used by the program (sample value to demonstrate
  gated access).
#### Usability
1. There can only be one admin authority (signer) at a time.
#### Supportability
1. Demonstrated as a sample app; the program's core functionality does
  not matter — the focus is on the admin authority pattern.
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

## ⏱ Timeline Expectations

Estimated duration: **2 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 dual License**.


## Resources

TODO: LEE official doc

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions,
please use **Discussions**.
