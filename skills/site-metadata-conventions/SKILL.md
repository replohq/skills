---
name: site-metadata-conventions
title: Site Metadata & SEO
summary: Set a site's title, description, social preview image, and search visibility.
description: Use when the user asks to change the site's title, description, OG image, social preview, favicon, or search-engine visibility (noindex), or any field they describe as "site settings". Triggers include "update the OG image", "change the site title", "set the description", "fix the social preview", "hide from Google", "add a favicon".
tools: upload_asset
---

# Site Metadata Conventions

Replo sites store all "site settings" — title, description, social preview image, favicon, and search-engine visibility — as Next.js App Router metadata in **one** file. This skill is the source of truth for how those edits are made.

## Where metadata lives

The metadata file is the site's **root layout** at `app/layout.tsx`, in the site
repo you cloned. Edit that file; do not create or author files under `src/app/`.

The root layout contains a top-level export:

```tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  // ...
};
```

Rules:

- All metadata edits go in this single object.
- If `metadata` doesn't exist yet, create it. Add the `import type { Metadata } from "next";` import if it's missing.
- Preserve any existing keys you aren't asked to change.
- When asked to remove a field, **delete the key entirely** from the object — don't set it to `null` or `undefined`.
- Don't touch the JSX rendered by the layout. Don't modify any file other than `layout.tsx` for these edits.
- Never write `<head>` tags or `<meta>` tags by hand in JSX. App Router renders them from the `metadata` export.

> The same `layout.tsx` also hosts the tracking-scripts / cookie-consent layer (`<ReploScripts>`, the consent banner, and the `data-replo-consent-mode` attribute). Those are owned by the **`tracking-scripts`** skill — when adding, removing, or changing analytics/marketing scripts or consent behavior, follow that skill and leave the consent layer untouched during metadata edits.

## Title

When the user changes the site title, set **all three** of these:

- `metadata.title`
- `metadata.openGraph.title`
- `metadata.twitter.title`

Mirroring across the three fields is mandatory — it's how social previews stay in sync with the browser title.

If the project uses a title template (e.g. `"%s | My Store"`), use the object form:

```tsx
metadata.title = { default: "<current title>", template: "<template>" };
```

Otherwise `metadata.title` is a plain string.

## Description

When the user changes the description, set **all three**:

- `metadata.description`
- `metadata.openGraph.description`
- `metadata.twitter.description`

Same mirroring rule as title.

## OG image (social preview)

The OG image goes in **two** places, both as **one-element arrays**:

- `metadata.openGraph.images = ["<url>"]`
- `metadata.twitter.images = ["<url>"]`

Never set them to a bare string. Next.js accepts strings here, but Replo always uses the array form for consistency.

### Uploading a new OG image

When the user provides a local image file (drag-and-drop, file path, etc.):

1. Call the `upload_asset` tool with the image.
2. Read the `url:` line from the tool output. That's the hosted URL.
3. Use that URL as the `[<url>]` value in both `metadata.openGraph.images` and `metadata.twitter.images`.

**Never invent or guess a URL.** If `upload_asset` fails, surface the error — don't fall back to a fabricated path. Don't commit images directly to the repo (e.g. `app/opengraph-image.png`); always go through `upload_asset` so the bytes land in R2 with a stable hosted URL.

## Twitter card

Whenever any of title, description, or OG image is set, ensure:

```tsx
metadata.twitter.card = "summary_large_image";
```

When the user removes all three (title, description, and OG image are all gone), remove the `metadata.twitter` key entirely.

## Search-engine visibility (noindex)

To hide the site from Google and other search engines:

```tsx
metadata.robots = { index: false, follow: false };
```

To make the site publicly indexable again, **remove the `metadata.robots` key entirely** — don't set it to any "public" value. The default behavior (no `robots` key) is what makes a site indexable.

## llms.txt

Every published site serves a machine-readable `/llms.txt` ([llmstxt.org](https://llmstxt.org)) — an LLM-oriented table of contents of the site's pages. **It is auto-generated on every publish** from each page's `export const metadata` (read statically from source — the humanized route as the link title, `metadata.description` as the blurb), excluding `noindex` pages, dynamic (`[slug]`) routes, and section routes. A localized site's `[lang]` segment is locale plumbing, not a dynamic route: `app/[lang]/x/page.tsx` is listed as `/x`. You do not write or maintain it, and it has no UI.

So the way to influence `/llms.txt` is the page metadata itself: give each page a specific `metadata.description`, and use the homepage's `metadata.title` / `description` (they become the file's H1 and summary). `noindex` pages drop out automatically.

**Escape hatch — a site can own its `/llms.txt`.** If the repo has a committed `public/llms.txt`, publish serves that verbatim and never regenerates over it. Only create one when the user explicitly wants a hand-curated `/llms.txt` (e.g. a custom intro or an `## Optional` section) — and tell them it then stops auto-updating, so new pages won't appear until they edit it. Otherwise leave the file absent and let publish generate it. An **empty** committed `public/llms.txt` is how Site Settings turns the feature off — don't "fix" one by deleting it unless the user asks for llms.txt back.

## robots.txt and sitemap.xml

These are ordinary Next metadata routes committed under `app/`. Publish scaffolds the default `robots.ts` and `sitemap.ts` when no owner exists.

`/robots.txt` has exactly **one** owner file. Never leave both owners in the site:

- **Auto (default):** `app/robots.ts`. To change crawler rules, flip the booleans in `export const seo` and change nothing else. Any other edit makes Site Settings treat the file as hand-owned and stop offering the toggles.
- **Custom:** `app/(replo-seo)/robots.txt/route.ts`. Edit only the JSON string assigned to `customRobotsRules`, which is served as the site's custom crawler rules. Use it when the user wants rules the toggles can't express, such as per-path `Disallow`, crawl delays, or extra user agents. The `// replo-managed-route:` marker line is the ownership contract: while it is present, Site Settings rewrites the whole file on save, so any other edit is overwritten. Delete the marker line only when the user wants to hand-own the file in code.

Never put a `Sitemap:` directive inside `customRobotsRules`. The generated Route Handler appends the current published domain's `/sitemap.xml` address when `app/sitemap.ts` exists, keeping the Sitemap toggle as the single source of truth and preventing stale domains.

The managed Auto file contains:

```ts
export const seo = {
  allowSearchEngines: true,
  allowAiAnswerCrawlers: true, // OAI-SearchBot, ChatGPT-User, PerplexityBot
  allowAiTrainingCrawlers: false, // GPTBot, ClaudeBot, CCBot, Google-Extended
  sitemap: true, // whether robots.txt advertises /sitemap.xml
};
```

Switch modes by swapping files in the same change:

- **To Custom:** create `app/(replo-seo)/robots.txt/route.ts` with the requested rules in `customRobotsRules`, then delete `app/robots.ts`.
- **Back to Auto:** create `app/robots.ts` from the managed scaffold, then delete `app/(replo-seo)/robots.txt/route.ts`.

Never let multiple owner files exist. The Next build fails and publish refuses the conflict. Never use `app/robots.txt` or `public/robots.txt` for new custom rules; those static paths are supported only so existing sites can migrate safely.

**`app/sitemap.ts` existing is the sitemap's on/off switch** — Next has no config flag for it. Deleting the file disables `/sitemap.xml`; publish won't scaffold it back once the deletion is committed. The route enumerates pages from the app directory at build time and skips `noindex` ones, so it needs no maintenance when pages are added. On a localized site, `[lang]` is treated as locale plumbing, not a dynamic segment: `app/[lang]/x/page.tsx` appears as `/x`.

The Auto route, Custom route, and sitemap route read `process.env.REPLO_SITE_URL` for the site's canonical origin, which publish injects. Absent it (e.g. a local build) they degrade quietly: no `Sitemap:` line, empty sitemap. Don't hardcode a domain in its place.

## Favicon

Favicons follow the same upload flow as OG images. The browser-tab icon always lives at **`metadata.icons.icon`** — never as a bare string on `metadata.icons`. Replo supports an optional dark-theme variant via `prefers-color-scheme` media queries, so the shape varies by what's set:

1. Call `upload_asset` with the file's path (once per variant).
2. Read the `url:` line from each tool output.
3. Write `metadata.icons.icon` using the shape that matches what's set:

| What's set       | `metadata.icons.icon` shape                                                                                                      |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Light only       | `{ url: "<light-url>" }`                                                                                                         |
| Light + dark     | `[{ url: "<light-url>", media: "(prefers-color-scheme: light)" }, { url: "<dark-url>", media: "(prefers-color-scheme: dark)" }]` |
| Dark only (rare) | `[{ url: "<dark-url>", media: "(prefers-color-scheme: dark)" }]`                                                                 |

Rules:

- Always nest under `.icon` — `metadata.icons = "<url>"` and `metadata.icons = { icon: "..." }` (without the explicit object form) are wrong shapes that the Site Settings modal won't round-trip correctly.
- When removing the favicon entirely, delete `metadata.icons.icon`. If `metadata.icons` becomes empty as a result, delete `metadata.icons` too. Preserve sibling keys like `metadata.icons.apple` or `metadata.icons.shortcut` if they exist.
- For Apple touch icons or other icon types that aren't the standard browser-tab favicon, use the appropriate sibling key (`metadata.icons.apple`, etc.) — don't put them inside `metadata.icons.icon`.

Don't commit favicon files directly to the repo (e.g. `app/icon.png`, `app/favicon.ico`, `app/apple-icon.png`). Always go through `upload_asset` and reference the hosted URL via `metadata.icons.icon`.

## What not to do

- Don't write `<head>`, `<meta>`, `<link rel="icon">`, or other head-tag JSX in the layout. Use the `metadata` export.
- Don't generate or guess URLs for OG images or favicons. Always use a URL returned by `upload_asset`.
- Don't commit image binaries to the project repo as a substitute for uploading them. `upload_asset` is always the right path for OG images, favicons, and any other site-metadata binary.
- Don't put metadata in the root `page.tsx` for site-wide settings — `layout.tsx` is the site-wide root. Page-level overrides belong in their own page files, but "site settings" always means the root `layout.tsx`.
- Don't modify multiple files for a single metadata change. One tool call to edit the root `layout.tsx` is enough.
- Don't create `src/app/`. Site App Router files belong under `app/`.

## Quick reference

| User intent               | Where to edit                                                                                                                                   |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Site title                | `metadata.title` + `metadata.openGraph.title` + `metadata.twitter.title`                                                                        |
| Site description          | `metadata.description` + `metadata.openGraph.description` + `metadata.twitter.description`                                                      |
| OG / social preview image | `upload_asset` → `metadata.openGraph.images = ["<url>"]` + `metadata.twitter.images = ["<url>"]` (+ `twitter.card = "summary_large_image"`)     |
| Hide from Google          | `metadata.robots = { index: false, follow: false }`                                                                                             |
| Make indexable            | Remove `metadata.robots` entirely                                                                                                               |
| Favicon (light only)      | `upload_asset` → `metadata.icons.icon = { url: "<url>" }`                                                                                       |
| Favicon (light + dark)    | `upload_asset` (×2) → `metadata.icons.icon = [{ url, media: "(prefers-color-scheme: light)" }, { url, media: "(prefers-color-scheme: dark)" }]` |
