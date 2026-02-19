---
id: RFP-002
title: Freeze Authority
tier: XS
funding: $XXXXX
status: open
category: Developer Tooling & Infrastructure
---


# RFP-002 — Freeze Authority

## 🧭 Overview

Build a proof-of-concept that demonstrates the ability to *freeze* an LEE
program, disabling all (or selected) interactions. This acts as a circuit
breaker; an emergency stop mechanism that an authorised account can trigger
when an issue is discovered.

This is particularly useful during early-stage program development and testing,
where the ability to halt interactions quickly can prevent exploits from causing
lasting damage.

This RFP targets a standalone PoC to validate the approach. A follow-up RFP
will be issued to package the pattern into a reusable developer library for
the Logos ecosystem.

## 🔥 Why This Matters

As the Logos ecosystem grows, programs deployed on LEZ need foundational
security primitives from day one. Without standardised emergency stop mechanism,
every team must design their own — leading to inconsistent implementations,
duplicated effort, and a higher risk of critical vulnerabilities.

Establishing these patterns early lowers the barrier for developers building
on LEE. Teams can focus on application logic rather than re-inventing admin
and freeze authority patterns, accelerating the pace at which new programs
ship. A proven PoC also gives the ecosystem a reference implementation that
builds confidence among developers and users alike — demonstrating that LEE
programs can be managed and secured with the same rigour expected in mature
smart contract environments.

This is a prerequisite for safe experimentation: as more programs are deployed
and begin handling real value, the ability for an authority to freeze a
compromised program is the difference between a contained incident and a
catastrophic loss.

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
#### Supportability
1. Demonstrated as a sample app; the program's core functionality does
  not matter — the focus is on the freeze pattern.

### Soft Requirements
If possible.
#### Reliability
1. Freeze authority can only be set to a valid new signer (on-curve key
  or deployed PDA), when set or initialised.

### Out-of-scope

Note that due to the privacy properties of LEE, it is not possible to freeze actions for a given signer/account. Hence, such a feature is not expected. Freezing is expected to be agnostic to the signers or initiators of a transaction.
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
