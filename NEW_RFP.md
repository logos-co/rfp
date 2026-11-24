# How to add a new RFP

1. Create a new file in `rfps/` named `RFP-###-short-title.md` based on
   [this template : RFP-000](/RFPs/RFP-000-template.md)
2. Include YAML frontmatter at the top like:

```yaml
---
id: RFP-001
title: Example Title
tier: XS/S/M/L/XL
status: open
category: Applications and Integrations
security_tier: 0/1/2/n/a
dependencies:
  - id: RFP-XXX
    reason: short reason this RFP depends on it
  - id: LP-XXXX
    reason: short reason this RFP depends on it
---
```

The `dependencies` field is a structured, machine-readable list of other RFPs,
Lambda Prizes (LP), platform features, or R&D items (tracked in
[logos-co/journeys.logos.co](https://github.com/logos-co/journeys.logos.co))
that must complete or exist before this RFP can be contracted or delivered. Use
canonical IDs (e.g. `RFP-XXX`, `LP-XXXX`). If the RFP is standalone, use an
empty list: `dependencies: []`.

Only **hard dependencies** belong in the frontmatter: blockers the RFP cannot be
contracted or delivered without. Each frontmatter entry must also be explained
in the body, in the Platform Dependencies section. Soft dependencies
(nice-to-have, stubbable with a placeholder, or merely better once available)
are described in the body only. When a dependency is delivered, keep its entry
for traceability.

The `security_tier` field sets which
[Security and Release Integrity](/README.md#security-and-release-integrity)
requirements apply: `0` for no secrets or user data, `1` for user data, `2` for
keys or funds (including wallets and AI agents that hold or can move funds). Use
`n/a` only when the RFP delivers no Logos Basecamp module. Reference the tier in
the RFP instead of restating its rules.

3. Include the frontmatter content on the [README.md](/README.md) file in the
   RFPs table
4. If the RFP is `open`, add it as an option in the **RFP ID** dropdown of
   [`.github/ISSUE_TEMPLATE/proposal.yml`](/.github/ISSUE_TEMPLATE/proposal.yml),
   e.g. `- "RFP-021 — Your Title"`. This drives the per-RFP label that proposal
   issues are tagged with (used for proposal counts in the BI dashboard).
5. Create a Pull Request and wait for review by the Logos EcoDev team
