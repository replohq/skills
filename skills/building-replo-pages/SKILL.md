---
name: building-replo-pages
title: Build Pages
summary: Create and edit pages backed by real store data, not hardcoded values.
description: "Use when creating, editing, extending, or restyling any page or section of a Replo site — landing pages, product and collection pages, layouts, styling, images, translations, or routing. Replo sites are Next.js App Router repos you clone and edit. Triggers: make me a page, create a landing page, build a product page, update the about page, add a section, restyle this, make this a component, translate my site. To deploy, use the publish skill."
tools: list_projects, list_sites, create_api_key, publish_site, find_assets, start_agent_session
---

# Building Replo Pages

A Replo site is a publishable Next.js App Router app. This is the operating
manual for building, editing, and extending its pages.

Two modes with opposite stances:

- **Building/editing pages — act autonomously.** Infer reasonable defaults; do
  not stop to ask for approval.
- **Publishing — never autonomous.** Deploy only on an explicit request. See the
  **publish** skill.

On anything that sells, you are the brand's conversion lead as well as its
builder, and you build for a **cold visitor**: someone who has never heard of
this brand and is mildly skeptical. Introduce the brand as if for the first
time, put proof before the first ask, and keep a specific risk-reversal beside
every buy decision. Invented proof stops at generic customer voices a merchant
can replace — never invent named press, celebrities, awards, certifications,
clinical claims, or aggregate review counts; those appear only when the user's
brief states them.

## Working on a Replo site from your own machine

1. Resolve the site: `list_projects` → `list_sites` (use the default site unless
   the user names one). Each site returns a `clone_url`.
2. Mint a key with `create_api_key` (include `repo.write` when you will push)
   and clone per the [Replo Git docs](/git/get-started).
3. `pnpm install && pnpm dev` gives a real local dev server — the Definition of
   Done checks and the visual audit below both run against it.
4. The Replo agent is a second writer of the same repo — pull before editing,
   push when done, never force-push.
5. Publish with `publish_site` only when the user explicitly asks.

## Critical defaults (read fully)

- **The app-router root is `app/`.** Keep every App Router file under `app/`. Do
  not create or author files under `src/app/`.
- **Stack:** Next.js App Router + TypeScript + Tailwind. `page.tsx` /
  `layout.tsx` are **Server Components** — push `"use client"` to the smallest
  leaf that needs interactivity, never onto a page or layout.
- **Tokens only — no raw color.** No hex/`rgb()`/`hsl()`/`oklch()`, no arbitrary
  Tailwind values (`bg-[#0a0a0a]`), and **no built-in palette classes**
  (`bg-slate-800`, `text-blue-500` are un-themeable too). Source color from
  semantic tokens. See `references/styling-tokens.md`.
- **Match the page's light/dark lane.** When adding or adapting a section on an
  existing page, inspect whether that page is visually dark or light and
  normalize the section into that lane. See `references/styling-tokens.md`.
- **Start from proven structure, not from memory.** Replo maintains a library of
  designed page and section templates; pages composed from it look intentional
  rather than like generic AI layouts. See "Sourcing page structure" below.
- **Never remove the runtime layer in `app/layout.tsx` when present:**
  `ReploProvider`, `<ReploScripts>`, or the `data-replo-consent-mode` attribute
  on `<html>` — these are cart/analytics/consent plumbing.
- **`/llms.txt` is auto-generated on publish — don't hand-maintain it.** Influence
  it via good page `metadata` (title + description) and `noindex`. See the
  **site-metadata-conventions** skill.
- **Default to creating a new page**, not overwriting one, unless the user
  explicitly says to replace a specific existing page.
- **Show progress early (hard rule).** For a new page, your **first write to the
  route must be the minimal skeleton — never the full page**, even when the page
  is small enough to one-shot. Confirm the route renders, share the local URL,
  then compose sections — and keep the page rendering after every subsequent
  edit. Writing the finished page as the first write is a defect.
- **Verify with the gates below + clean TypeScript.** You do not need a
  production build to verify; `publish_site` runs the real build.

## Sourcing page structure

Replo's template library is the cheap path to solid structure, layout rhythm,
and interaction quality. It is served by the Replo agent rather than as a
direct tool, so compose from it by prompting a session:

> `start_agent_session`: "Compose a `<page type>` page for `<brand or vertical>`
> using the Replo template library — I want the section structure and real
> source, not a description."

Then pull the result into your clone and adapt it: preserve the quality-bearing
contract (responsive intent, accessibility, interaction wiring, aspect-ratio and
media strategy) while normalizing color→tokens, light/dark lane, spacing, type
roles, CTA treatment, copy, and imagery so independent sections read as one
coherent page.

When you do write a section directly, hold it to the same bar: a distinct
conversion job, real copy rather than filler, and the slop checks in
`references/building-pages.md`.

## Working on a page (the spine)

Details live in `references/building-pages.md`; this is the loop.

**Steps 2–3 are the commerce path** — they fire when the page's goal is selling
or routing to selling. Non-commerce work (SaaS marketing, agency or portfolio
sites, docs) skips them and the conversion check; every other rule still applies.

1. **Inspect context and decide the job.** For edits, read the existing route and
   components first. For new pages, decide page type, audience, offer, objection,
   and primary action, then the visual lane and token direction.
2. **Classify the business** (commerce path). Determine the **vertical**; per
   product the **price tier** (impulse under ~$50, considered ~$50–150,
   high-ticket above ~$150), the **purchase shape** (`replenishable/consumable`,
   `multi-unit plausible`, `kit/bundle`, `sized-or-fitted`, `one-time durable`),
   and variants; then the SKU count and the page inventory the site needs. Use
   the routes a merchant expects: `/`, `/products/<slug>`, `/collections/<slug>`,
   `/pages/<slug>`, `/blog/<slug>`. **Every site gets a homepage at `/`** if one
   does not already exist. Never rewrite an existing homepage without explicit
   instruction.
3. **Post the Conversion Brief before the first page write** (commerce path). A
   short visible message, under ~20 lines per page: page type; the single primary
   action; the top three objections a cold visitor will have and which section
   answers each; the section list with each section's conversion job; the offer
   structure you chose and why. An open brief ("use your judgement") is
   permission to construct a real offer, not to skip one.
4. **Source structure**, then fill it section by section (see above).
5. **Check the user's real assets early** with `find_assets` (omit the query to
   list the full library).
6. **Gate each section as you finish it** — don't defer to an end-of-page pass —
   then run the conversion check and the Definition of Done before declaring
   complete.

## UI component library

A full shadcn library on Base UI lives in `components/ui/` — prefer it over hand-rolled HTML/Tailwind. List `components/ui/` in the repo to see what is available.

Import via `@/components/ui/...`, merge classes with `cn()` from `@/lib/utils`, icons from `lucide-react`. Compose forms from the installed input/label/select/checkbox/radio/field components. For toasts, use `sonner` only when `<Toaster />` from `@/components/ui/sonner` is mounted in `app/layout.tsx`. The `@/*` alias maps to the project root.

## Site Components

Components reused across pages live in the site repo's top-level `components/` directory (create it when absent), imported through `@/components/...`. Every top-level `.tsx` file there is surfaced to the user by file name as a Site Component in Site Builder — a product surface with isolated preview and a generated props panel, mentionable in chat. Hard rules: **one standalone file per component** — default export, subcomponents and helpers inlined, no sibling helper files, and no imports of other site files (npm packages, `@/components/ui/*`, `@/lib/utils`, and type-only `@replohq/sdk` imports are fine); every prop serializable and defaulted so the component renders bare. Do not create or link a separate components repository. For the full contract, prop design for the props panel, promoting existing page markup into a component ("use my same header on this page", "make this a component"), and preview recovery, read `references/site-components.md`.

## Visual audit pass (required before finishing)

After the conversion check, view every built page with your browser tooling at BOTH a phone-size viewport and a desktop-size viewport (for example 390×844 and 1440×900) and fix what you see before finishing.
On the phone pass: on `homepage`/`product`/`landing-offer` pages the hero's
primary CTA fully visible without scrolling; exactly one button color reading
as "the" action (everything else outline/secondary/ghost); no broken images,
overlaps, or horizontal overflow; a page that reads as a real brand rather than
a template. On the desktop pass, hunt phone patterns leaking into the wide
rendering: a sticky mobile cart bar showing where the buy box is already
visible, a single-column stack where the content supports side-by-side columns,
buttons stretched far wider than their label wants, text running unconstrained
line lengths, imagery composed only for a narrow screen. State in the final
summary that both passes ran and what each fixed — a page you have not seen at
both sizes is not done.

## Definition of Done

Run from the site repo root. Any output from checks 1–5 is a defect — fix it and re-run; check 6 is a review list read against the rule stated under it. (`grep` is portable; `rg` works too.)

```bash
SRC="app components"

# 1) Raw colors in components (color belongs only in globals.css)
grep -rEn --include='*.tsx' --include='*.ts' \
  -e '#[0-9a-fA-F]{3,8}\b' -e '\b(rgba?|hsla?|oklch|oklab|lab|color-mix)\(' $SRC | grep -v globals.css

# 2) Arbitrary values baked into className: [#...], [12px], [1.5rem]
grep -rEn --include='*.tsx' -e '\[#' -e '\[(rgb|hsl|oklch)' -e '-\[[0-9.]+(px|rem|em)\]' $SRC

# 3) Un-themeable built-in palette classes (the check most agents miss)
grep -rEn --include='*.tsx' \
  -e '\b(bg|text|border|ring|from|via|to|fill|stroke|decoration|outline|divide|placeholder|caret|accent)-(slate|gray|zinc|neutral|stone|red|orange|amber|yellow|lime|green|emerald|teal|cyan|sky|blue|indigo|violet|purple|fuchsia|pink|rose)-(50|100|200|300|400|500|600|700|800|900|950)\b' \
  -e '\b(bg|text|border|ring|from|via|to|fill|stroke|decoration|outline|divide|placeholder|caret|accent)-(black|white)\b' $SRC

# 4) Server/client boundary: no "use client" in a page or layout
grep -rEn --include='page.tsx' --include='layout.tsx' -e 'use client' app

# 5) No raw <img> (use next/image)
grep -rEn --include='*.tsx' -e '<img\b' $SRC

# 6) Primary-fill buttons (Button defaults to the primary variant when none is passed)
grep -rEn --include='*.tsx' -e '<Button\b' $SRC | grep -vE 'variant=[{"]*"?(outline|secondary|ghost)'
```

Every line check 6 prints must be an instance of that page's one primary action — repeats of the same CTA down a long page are expected. Anything else (nav, cards, quantity steppers, secondary links) is a reserved-color defect; give it `variant="outline"`, `"secondary"`, or `"ghost"`. A `<Button` whose `variant` sits on a later line or behind a helper call prints too — read the tag before treating it as a hit.

Then confirm: **LSP/TypeScript is clean**, **no skeleton placeholder blocks remain** (a leftover placeholder is a defect the same way filler copy is), every `<Image>`/`next/image` has explicit dimensions and meaningful `alt`, and the page passes a quick **slop check** (no centered-gradient hero with no product, no reusable black/purple SaaS shell when the prompt implies a different world, no text-only first viewport where stats are the only proof, no lone row of 3 identical icon cards, every section has a distinct conversion job, real product copy not filler). The slop catalog and per-section checklist are in `references/building-pages.md`.

### Conversion check (every commerce page)

Re-read the page's Conversion Brief and verify the page did every conversion job the brief listed (jobs, not literal order — see step 4) and shipped every rule in the `cro-universal-rules` guidance, the offer structure `cro-offer-structure` prescribes for this product's purchase shape, and every non-negotiable from its vertical file and its page-type file. Fix the gaps rather than reporting them. Two gates are mandatory and are defects when they fail:

- **Fold CTA** — on the page types whose hero carries the primary action (`homepage`, `product`, `landing-offer`), the hero's primary CTA button is fully visible inside the first mobile viewport (390×844) with no scrolling. `article` and `about` defer the ask on purpose, so their substitute is the first CTA appearing within the first two viewports; on `collection`, the product cards are the actionable surface, so the first card is what must appear that fast. Verify visually at 390×844 with Playwright when browser tooling is available; otherwise verify compositionally — the hero's headline + subline + CTA block is compact enough to clear the fold above a tall hero image. This constrains the phone rendering only; the desktop rendering keeps its own native composition.
- **Reserved primary color** — the primary button color is reserved for the page's one primary action alone; repeated instances of that same CTA down the page are expected and correct. Every other button (nav, cards, quantity steppers, secondary links) uses `variant="outline"`, `"secondary"`, or `"ghost"`. Definition of Done check 6 lists the candidates.

Confirm both in one line per page in your final summary.

### Verifying dynamic routes

A page like `app/products/[slug]/page.tsx` is a dynamic page (one page rendering many URLs); the literal bracket URL (`/products/[slug]`) is not a real page. Never use it to verify.

- On the literal bracket URL, a **404** (pages that look params up in local data) or a **"Sample product" render** (pages built on canopy loaders, which substitute sample data on dev servers) is expected preview scaffolding — not a defect. Don't fix it, don't report it as a problem, and ignore matching bracket-URL 404 lines in dev-server logs: the editor's preview probes that URL itself.
- A "Sample product" render also does **not** prove real data works. Verify with a concrete URL: take a real value from the page's `generateStaticParams`, the site's data files, connected integration data, or a link on a listing page (e.g. `/products/blue-hat`), then fetch/open that.
- For pages backed by local data files, export `generateStaticParams()` listing valid params — it documents what the page accepts, makes verification one file-read away, and lets the preview toolbar enumerate real values into a dropdown. Keep the data in a **plain data module that exports a literal typed array** (e.g. `data/posts.ts` → `export const posts = [{ slug: "launch-day", ... }]`) and have `generateStaticParams` derive from it via `.map` (`return posts.map((post) => ({ slug: post.slug }))`). The preview reads this statically, so avoid computing params from fetches, filesystem reads, or transforms — those force a slower LLM guess. Skip `generateStaticParams` entirely for integration-backed pages (a build-time fetch can slow or break publish).

### Conversion check (every commerce page)

Re-read the page's Conversion Brief and verify the page did every conversion job the brief listed (jobs, not literal order) and shipped a coherent offer for this product's purchase shape. Fix the gaps rather than reporting them. Two gates are mandatory and are defects when they fail:

- **Fold CTA** — on the page types whose hero carries the primary action (`homepage`, `product`, `landing-offer`), the hero's primary CTA button is fully visible inside the first mobile viewport (390×844) with no scrolling. `article` and `about` defer the ask on purpose, so their substitute is the first CTA appearing within the first two viewports; on `collection`, the product cards are the actionable surface, so the first card is what must appear that fast. Verify visually at 390×844 when browser tooling is available; otherwise verify compositionally — the hero's headline + subline + CTA block is compact enough to clear the fold above a tall hero image. This constrains the phone rendering only; the desktop rendering keeps its own native composition.
- **Reserved primary color** — the primary button color is reserved for the page's one primary action alone; repeated instances of that same CTA down the page are expected and correct. Every other button (nav, cards, quantity steppers, secondary links) uses `variant="outline"`, `"secondary"`, or `"ghost"`. Definition of Done check 6 lists the candidates.

Confirm both in one line per page in your final summary.

### Verifying dynamic routes

A page like `app/products/[slug]/page.tsx` is a dynamic page (one page rendering many URLs); the literal bracket URL (`/products/[slug]`) is not a real page. Never use it to verify.

- On the literal bracket URL, a **404** (pages that look params up in local data) or a **"Sample product" render** (loaders substitute sample data on dev servers) is expected preview scaffolding — not a defect. Don't fix it, don't report it as a problem, and ignore matching bracket-URL 404 lines in dev-server logs.
- A "Sample product" render also does **not** prove real data works. Verify with a concrete URL: take a real value from the page's `generateStaticParams`, the site's data files, connected integration data, or a link on a listing page (e.g. `/products/blue-hat`), then fetch/open that.
- For pages backed by local data files, export `generateStaticParams()` listing valid params — it documents what the page accepts, makes verification one file-read away, and lets the preview toolbar enumerate real values into a dropdown. Keep the data in a **plain data module that exports a literal typed array** (e.g. `data/posts.ts` → `export const posts = [{ slug: "launch-day", ... }]`) and have `generateStaticParams` derive from it via `.map` (`return posts.map((post) => ({ slug: post.slug }))`). The preview reads this statically, so avoid computing params from fetches, filesystem reads, or transforms — those force a slower LLM guess. Skip `generateStaticParams` entirely for integration-backed pages (a build-time fetch can slow or break publish).
