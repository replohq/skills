---
name: replo-branding
title: Create a Brand Kit
summary: Extract a brand from a website or Figma file, then keep it updated.
description: "Use when creating or updating a Replo brand kit — extracting a brand from a website URL, handling a Figma URL as the brand source, and applying the brand — all by prompting a Replo session."
tools: start_agent_session
---

# Brand Kits via Replo Sessions

A Replo project's brand — colors, fonts, logos, imagery, and a business profile — lives in **Brand Studio**. There is no dedicated brand tool on the public MCP surface, and none is needed: every brand operation works today by describing it in a session (`start_agent_session` / `send_agent_message`). Never tell a user that brand kits need "write access" or are unsupported through the connector — they are fully operable.

## Create a brand kit from a website URL

This is the most complete path. The Replo agent scrapes the site, extracts the palette, fonts, logo, and imagery, writes the business profile, and saves the result in Brand Studio.

Prompt shape:

```text
Create a brand kit for this project from https://example.com
```

Notes:

- Name the project (the session is already project-scoped, so the URL is the only required input).
- The project's **first** brand becomes the primary brand automatically — no extra step before it can be applied to a site.
- Extraction takes a minute or two: poll `get_agent_session` while status is `starting`/`running`, and give the user the `dashboardUrl` so they can watch the brand stream into Brand Studio.

## Create or update a brand without a URL

The agent can also build a brand from a conversation (no scrapeable site) or edit an existing one surgically:

```text
Walk me through setting up my brand — I don't have a website yet.
```

```text
Change the brand's primary color to teal and swap the heading font to Playfair Display.
```

Brand edits preserve everything not mentioned — the agent mutates only the named tokens.

## Figma URL as the brand source

If the user offers a Figma file as the brand source, pass the URL to a session:

```text
The brand should come from this Figma file: <figma-url>. Check whether the Figma integration is connected and tell me what you can extract.
```

- The session checks the project's Figma integration. **If it is not connected**, the session says so — relay that to the user and have them connect Figma in the Replo dashboard's Integrations app, then continue.
- Extraction from a website URL is the more complete brand path today. When the user has a live site, offer that first; the Figma file works best as a supplement (specific colors, type styles, or assets the user points at).
- Do not promise a full automatic Figma → brand-kit extraction; state what the session reports it can do.

## Apply the brand

Applying the brand to a site is a design-token change on that site, not a rebuild:

- **Via session** (default): `Apply the project's brand to the <site> site.` The agent rewrites the site's design tokens, font, and logo, and runs its own contrast and coverage checks.
- **Via local edit**: if you are already working in a clone of the site repo, follow the [Applying a brand to a site](/mcp/skills/apply-branding) skill instead.

## Reading the brand

To use brand values yourself (for ads, emails, or local site edits), ask a session to report them:

```text
Report the project's primary brand: every color token with its hex, the font families and faces, and the logo URL. Do not change anything.
```
