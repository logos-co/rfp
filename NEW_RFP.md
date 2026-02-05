# How to add a new RFP

1. Create a new file in `rfps/` named `RFP-###-short-title.md` based on [this template : RFP-000](/RFPs/RFP-000-template.md)
2. Include YAML frontmatter at the top like:

```yaml
---
id: RFP-001
title: Example Title
tier: M
funding: $5,000–$15,000
status: open
category: Applications and Integrations
---
```

3. Include the frontmatter content on the [README.md](/README.md) file in the RFPs table
4. Create a Pull Request and wait for review by the Logos EcoDev team
