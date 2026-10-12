---
name: replo-branding
title: Create a Brand Kit
summary: Extract a brand from a website or Figma file, then keep it updated.
description: "Use when creating or updating a Replo brand kit — extracting a brand from a website URL, handling a Figma URL as the brand source, and applying the brand — through Replo’s brand and page tools."
tools: get_brand_kit, create_or_update_brand_kit, apply_brand_kit_to_site, get_agent_session
---

# Brand Kits via Replo Sessions

A Replo project's brand — colors, fonts, logos, imagery, and a business profile — lives in **Branding**. Use `create_or_update_brand_kit` for requested brand changes and the same dedicated tool with its `sessionId` for follow-ups within that operation. Supply the projectId and only the necessary brand instructions and references. Use `get_brand_kit` for reads. Applying the saved brand to a site is a separate requested change through `apply_brand_kit_to_site`, with the siteId from `list_sites`.

## Create a brand kit from a website URL

This is the most complete path. The Replo agent scrapes the site, extracts the palette, fonts, logo, and imagery, writes the business profile, and saves the result in Branding.

Prompt shape:

```text
Create a brand kit for this project from https://example.com
```

Notes:

- Pass the selected projectId and the website URL in instructions to `create_or_update_brand_kit`.
- The project's **first** brand becomes the primary brand automatically — no extra step before it can be applied to a site.
- Extraction takes a minute or two: poll `get_agent_session` while status is `starting`/`running`, and give the user the `dashboardUrl` so they can watch the brand stream into Branding.

## Create or update a brand without a URL

The agent can also build a brand from a conversation (no scrapeable site) or edit an existing one surgically:

```text
Walk me through setting up my brand — I don't have a website yet.
```

```text
Change the brand's primary color to teal and swap the heading font to Playfair Display.
```

Request only the named changes and verify that the result preserves other brand values.

## Figma URL as the brand source

If the user offers a Figma file as the brand source, follow the Figma skill and pass the original URL and requested brand details to `create_or_update_brand_kit`:

```text
The brand should come from this Figma file: <figma-url>. Check the project’s Figma connection and extract the requested colors, typography, and assets.
```

- The Replo agent uses the project’s Figma integration. If it needs a connection, relay its setup link and continue the same operation after the user connects.
- Extraction from a website URL is the more complete brand path today. When the user has a live site, offer that first; the Figma file works best as a supplement (specific colors, type styles, or assets the user points at).
- Do not promise a full automatic Figma → brand-kit extraction; state what the session reports it can do.

## Apply the brand

Applying the brand to a site is a design-token change on that site, not a rebuild:

- **Via `apply_brand_kit_to_site`** (default): pass the selected siteId and the requested brand attributes. The agent changes shared styling, fonts, and logo references. Report checks it could not run; do not imply a build or visual pass occurred.
- **Via local edit**: if you are already working in a clone of the site repo, follow the **apply-branding** skill instead.

## Reading the brand

To use brand values yourself (for ads, emails, or local site edits), call `get_brand_kit` with the projectId. It reads the saved brand without starting an agent session or making changes.
