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
---
```

3. Include the frontmatter content on the [README.md](/README.md) file in the
   RFPs table
4. If the RFP is `open`, add it as an option in the **RFP ID** dropdown of
   [`.github/ISSUE_TEMPLATE/proposal.yml`](/.github/ISSUE_TEMPLATE/proposal.yml),
   e.g. `- "RFP-021 — Your Title"`. This drives the per-RFP label that proposal
   issues are tagged with (used for proposal counts in the BI dashboard).
5. Create a Pull Request and wait for review by the Logos EcoDev team
