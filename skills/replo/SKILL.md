---
name: replo
description: "Use when the user wants to start or run a business online through the Replo connector: build and publish a website or page with Site Builder, then sell products, take orders, read analytics, connect a domain or research competitors' ads. Before starting, call the connector's skills tool to see the available skills, then read the ones you need. Do not trigger for unrelated website, analytics, or third-party service requests."
compatibility: Requires the Replo connector.
---

# Replo

Replo's skills live in the Replo connector's `skills` tool, so they stay current without changing this skill. Before doing Replo work, call `skills` to get the full list, then `skills` with a name to load the one you need.

1. Call `skills` with no arguments to list every skill: each one's name and description.
2. Load one skill's full `SKILL.md` with `skills({ name: "..." })`, starting with `replo`, which explains which tool does what. Make one call per skill.
3. Follow those skills and the connector's instructions.
