# How to add a new RFP

1. Create a new file in `rfps/` named `RFP-###-short-title.md` based on
   [this template : RFP-000](/RFPs/RFP-000-template.md)
2. Include YAML frontmatter at the top like:

```yaml
---
id: RFP-001
title: Example Title
tier: XS/S/M/L/XL
funding: $5,000–$15,000
status: open
category: Applications and Integrations
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

3. Include the frontmatter content on the [README.md](/README.md) file in the
   RFPs table
4. If the RFP is `open`, add it as an option in the **RFP ID** dropdown of
   [`.github/ISSUE_TEMPLATE/proposal.yml`](/.github/ISSUE_TEMPLATE/proposal.yml),
   e.g. `- "RFP-021 — Your Title"`. This drives the per-RFP label that proposal
   issues are tagged with (used for proposal counts in the BI dashboard).
5. Create a Pull Request and wait for review by the Logos EcoDev team
