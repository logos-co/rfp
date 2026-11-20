---
id: RFP-000
title: Project Title
tier: XS/S/M/L/XL
status: open/closed
category: Developer Tooling & Infrastructure / Applications & Integrations / Ecosystem & Community Enablement
# Security tier, as defined in README "Security and Release Integrity":
# 0 = no secrets or user data, 1 = user data, 2 = keys or funds (including
# wallets and AI agents that hold or can move funds). Use n/a only when the
# RFP delivers no Logos Basecamp module.
security_tier: 0/1/2/n/a
dependencies:
  # Hard dependencies only: RFPs, Lambda Prizes (LP), platform
  # features, or R&D items (tracked in logos-co/journeys.logos.co)
  # that must complete or exist before this RFP can be contracted or
  # delivered. Use canonical IDs (RFP-XXX, LP-XXXX).
  # Keep entries after they are delivered for traceability. Each entry
  # must also be explained in the Platform Dependencies section of
  # the body. If the RFP is standalone, use an empty list:
  # `dependencies: []`. Soft dependencies (nice-to-have, stubbable, or
  # better-once-available) are described in the body only, not here.
  - id: RFP-XXX
    reason: short reason this RFP depends on it
  - id: LP-XXXX
    reason: short reason this RFP depends on it
---

<!-- Don't forget to add this RFP to the table in README.md (between RFP_TABLE_START / RFP_TABLE_END markers) -->

# RFP-000 — Project Title

> **Note.** This specification describes an outcome that may benefit the Logos
> ecosystem. It is a proposal rather than an instruction. Its requirements
> reflect the technical compatibility with the Logos technology stack and are
> the criteria against which proposals and milestones are evaluated. Logos makes
> no representation as to the legal or regulatory treatment of this
> specification or any implementation of it in any jurisdiction.
>
> Teams implementing it are solely responsible for (i) assessing the risks and
> implications of what they build; (ii) obtaining their own professional advice;
> and (iii) for complying with any legal and regulatory requirements that apply
> to them. Software developed under the Program is published and maintained by
> its developers, not by Logos.
>
> Anyone who chooses to deploy, host, operate or use software developed under
> the Program, whether or not they were awarded a grant under the Program, does
> so at their own risk and is solely responsible for complying with any legal or
> regulatory requirements that apply to them. See the
> [Terms & Conditions](../TERMS_AND_CONDITIONS.md).
>
> Deploying the software described in this RFP, operating any service based on
> it, or carrying on business through it may amount to regulated activity in
> some jurisdictions, including where it involves holding or managing users'
> assets or providing services to others. Whoever conducts any such activity
> does so as principal, in their own name, and is solely responsible for
> assessing its regulatory treatment, including any licensing, registration,
> sanctions or anti-money laundering obligations that may apply to them. Logos
> does not make any representation, provide any advice or assume any
> responsibility in respect of any such determination or compliance.

## 🧭 Overview

Briefly describe the opportunity in 4–6 sentences.

Explain things like:

- What needs to be built
- Why it matters
- How it strengthens the Logos ecosystem
- The type of team likely to succeed
- The LEZ testnet version the LEZ program is delivered for (currently testnet
  0.4)

## 🔥 Why This Matters

Provide ecosystem context in 4-6 sentences.

Explain things like:

- Why this RFP is critical in the success of the Logos ecosystem
- How can this RFP unlock user or developer adoption
- How can this RFP enable liquidity
- How can this RFP improve privacy guarantees
- How can this RFP reduce technical barriers

Builders want to understand the impact of th RFP towards the success of Logos
ecosystem.

## ✅ Scope of Work

### Hard Requirements

Use FURPS framework. Each numbered item should be a testable statement.

#### Functionality

List what the program/application must do.

#### Usability

Standard requirements for Logos apps (adapt as needed):

1. Provide an SDK that can be used to build Logos modules for interacting with
   the program.
2. Provide a Logos mini-app GUI with local build instructions, downloadable
   assets, and loadable in Logos app (Basecamp) via git repo.
3. Provide a CLI that covers core functionality of the program. The CLI may have
   fewer features than the GUI mini-app but must support all essential
   operations.
4. Provide an IDL for the LEZ program, preferably using the
   [SPEL framework](https://github.com/logos-co/spel).

Add RFP-specific usability requirements here.

#### Reliability

List reliability guarantees (consistency, fault tolerance, graceful degradation,
etc.).

#### Performance

List performance requirements. Document compute unit usage of each operation
(LEZ's per-transaction compute budget may change during testnet).

#### Supportability

Standard requirements (adapt as needed):

01. The LEZ program is delivered for LEZ testnet 0.4: it is deployed and tested
    on LEZ testnet 0.4.
02. End-to-end integration tests run against a LEZ sequencer (standalone mode)
    and are included in CI
03. CI must be green on the default branch.
04. Every hard requirement in Functionality, Usability, Reliability, and
    Performance has at least one corresponding test.
05. A README documents end-to-end usage: deployment steps, program addresses,
    and step-by-step instructions for interacting with the program via CLI and
    mini-app.
06. Provide documentation in repo for the Logos module/library, covering the
    developer integration journey.
07. Provide documentation in repo for the CLI, covering the core operator/user
    journey.
08. Provide Figma designs or equivalent for all GUI artifacts.
09. Provide a privacy and anonymisation properties document covering: what
    on-chain state and transaction data is visible to observers; what data is
    protected when the private account path is used; trust assumptions,
    specifying which guarantees are enforced by the on-chain program and which
    depend on correct client behaviour; and what happens if a user bypasses the
    expected interaction path.
10. Provide a design decision log in repo recording each major design decision
    and choice: the context, the options considered, the option chosen, and the
    rationale and trade-offs. The log is updated as decisions are made and is
    current at each milestone delivery.
11. Logos Basecamp modules are built with
    [`logos-module-builder`](https://github.com/logos-co/logos-module-builder)
    and published to a module catalog forked from
    [`logos-modules-release-base`](https://github.com/logos-co/logos-modules-release-base),
    hosted on a GitHub organisation account.
12. Releases follow
    [Security and Release Integrity](../README.md#security-and-release-integrity)
    for this RFP's `security_tier`: every `.lgx` is signed with `lgx sign` by a
    signer DID declared up front, published as an immutable GitHub release, and
    never replaced in place. Third-party GitHub Actions are pinned to commit
    SHAs.
13. Each milestone delivery includes a release integrity manifest (source commit
    SHA, catalog URL, package and version, release URL, `rootHash`, SHA-256,
    built variants, signer DID) and install instructions that give the exact
    catalog URL, signer DID and `rootHash`.
14. Build and demo scripts run without `--accept-flake-config`, extra Nix
    substituters, unpinned remote scripts or reviewer credentials.

#### Security

Reference the tier instead of restating its rules. Keep the item that matches
`security_tier` and remove the others:

- Tier 1: the README includes a data-handling note: what the module stores or
  sends, where, and how it is protected.
- Tier 2: the Tier 2 requirements in
  [Security and Release Integrity](../README.md#tier-2-requirements) apply: a
  threat model in the first milestone, approval enforced in the core module, no
  plaintext keys, spend caps enforced outside any agent, and an audit or
  independent security review milestone.

Add RFP-specific security requirements here (for example, which methods must
only be callable by declared modules).

### Soft Requirements

Explain all the optional soft requirements of the RFP in points.

## 👤 Recommended Team Profile

Signal the bar without gatekeeping. Mention all the areas where the applying
should ideally have experience with. Examples:

- Distributed systems
- Cryptography
- Wallet infrastructure
- Production deployments

## ⏱ Timeline Expectations

Provide a realistic range aligned with tier. Example: Estimated duration: **3–7
weeks**

## 🌍 Open Source Requirement

All code must be released under the **MIT+Apache2.0 License**.

## Resources

List all the relevant resources that can be useful to understand more about the
RFP/related work.

The Resources listed above, and any third-party material referred to elsewhere
in this specification, are for informational purposes only and are intended
solely as guidance regarding tools or other artefacts that may be relevant to
preparing a Submission under the Program. They may include, or may refer to,
earlier published specifications, some of which may have received Submissions.
Logos does not endorse or approve, and makes no representation regarding, any
Submission or referenced material. It is ultimately the Applicant’s
responsibility to determine whether, and how, if at all, to use any Resource.

## ✏️ How to Apply

👉 Submit a proposal using the Issue form:

**[Submit Proposal](https://github.com/logos-co/rfp/issues/new?template=proposal.yml)**

We typically respond within **14 days**. For clarification questions, please use
**Discussions**.
