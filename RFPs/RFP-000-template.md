---
id: RFP-000
title: Project Title
tier: XS/S/M/L/XL
funding: $XXXXX
status: open/closed
category: Developer Tooling & Infrastructure / Applications & Integrations / Ecosystem & Community Enablement
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-000 — Project Title

## 🧭 Overview

Briefly describe the opportunity in 4–6 sentences.

Explain things like:

- What needs to be built
- Why it matters
- How it strengthens the Logos ecosystem
- The type of team likely to succeed


## 🔥 Why This Matters

Provide ecosystem context in 4-6 sentences.

Explain things like:

- Why this RFP is critical in the success of the Logos ecosystem
- How can this RFP unlock user or developer adoption  
- How can this RFP enable liquidity  
- How can this RFP improve privacy guarantees  
- How can this RFP reduce technical barriers

Builders want to understand the impact of th RFP towards the success of Logos ecosystem.


## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

List what the program/application must do.

#### Usability

Standard requirements for Logos apps (adapt as needed):

1. Provide an SDK that can be used to build Logos modules for
   interacting with the program.
2. Provide a Logos mini-app GUI with local build instructions,
   downloadable assets, and loadable in Logos app (Basecamp) via
   git repo.
3. Provide a CLI that covers core functionality of the program.
   The CLI may have fewer features than the GUI mini-app but must
   support all essential operations.
4. Provide an IDL for the LEZ program, preferably using the
   [SPEL framework](https://github.com/logos-co/spel).

Add RFP-specific usability requirements here.

#### Reliability

List reliability guarantees (consistency, fault tolerance, graceful
degradation, etc.).

#### Performance

List performance requirements. Document compute unit usage of each
operation (LEZ's per-transaction compute budget may change during
testnet).

#### Supportability

Standard requirements (adapt as needed):

1. The program is deployed and tested on LEZ devnet/testnet.
2. End-to-end integration tests run against a LEZ sequencer (standalone
   mode) and are included in CI
3. CI must be green on the default branch.
4. Every hard requirement in Functionality, Usability, Reliability,
   and Performance has at least one corresponding test.
5. A README documents end-to-end usage: deployment steps, program
   addresses, and step-by-step instructions for interacting with the
   program via CLI and mini-app.

### Soft Requirements

Explain all the optional soft requirements of the RFP in points.


## 👤 Recommended Team Profile

Signal the bar without gatekeeping. Mention all the areas where the applying should ideally have experience with. Examples:

- Distributed systems  
- Cryptography  
- Wallet infrastructure  
- Production deployments


## ⏱ Timeline Expectations

Provide a realistic range aligned with tier. Example: Estimated duration: **3–7 weeks**


## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 License**.


## Resources

List all the relevant resources that can be useful to understand more about the RFP/related work.


## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use **Discussions**.
