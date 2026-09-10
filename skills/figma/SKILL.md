---
name: figma
title: Figma Designs
summary: Read a Figma file through the Replo connection and rebuild the design as a page with exact fonts, layout, colors, and assets.
tools: get_integration_status, upload_asset, find_assets, start_agent_session
description: "REQUIRED when the user wants to build, rebuild, recreate, replicate, or match a Figma design, frame, or page, or to correct the fonts, layout, spacing, or imagery of a page built from one. Triggers: a figma.com link, \"from Figma\", \"this Figma file\", \"match the Figma\". Also covers reading Figma files, nodes, components, and styles through the project's Figma connection."
---

# Building a Replo page from a Figma design

Rebuild the design in React + Tailwind using BOTH the rendered image (visual
truth) and the structured node data (layout + design system) — never eyeball
pixels alone, and never build from a pasted screenshot when the file itself is
reachable.

Figma is reached through Replo's own Figma tools, not through Figma's MCP
server or API. Each operation is its own tool, named `figma_<operation>` — the
ones below are in your tool list already.

This skill governs **fidelity**: what the source is, how to read it, and what
"done" means. The **building-replo-pages** skill still governs the repo: clone
the site, keep the App Router under `app/`, write the skeleton first and keep
the route rendering, use `next/image`, keep color in tokens, and run its
Definition of Done. Where the two disagree, the rules below win — a Figma build
is spec-driven, not brief-driven.

## Parse the link

Figma has no search or browse endpoint, so a team or project URL is useless.
Ask the user to link the specific **file**. The key is the URL segment after
`/design/` or `/file/`, and the target node is the `node-id` query parameter —
note it arrives URL-encoded (`9-64`) but the API wants `9:64`. Keep learned
file keys in whatever notes you keep for this project so the user is not asked
for the same link twice, and preserve the original URL in your summary even
when you resolved a different target node.

## Decide the mode before touching the design

Read every relevant user message and set three things. Do not add creative
direction, destination-brand substitutions, or content changes the user did
not request.

- **Fidelity: `faithful` (default) or `adapt`.** `adapt` is valid only when the
  user explicitly asked, in their own words, to rebrand, reinterpret, reskin,
  or replace source content. The destination site's business, brand kit,
  existing design tokens, or product catalog never authorize changing the
  design. When the user does ask to adapt, follow **apply-branding** for the
  token work rather than repainting components by hand.
- **Coverage: desktop and mobile (default).** Cover both even when the user
  says only "replicate this" and links one frame. If the file has
  corresponding desktop and mobile frames, both are binding. If it has one
  viewport, match it exactly and derive the other from the same content and
  visual system. Build a single viewport only when the user explicitly asks
  for that.
- **Font policy: exact.** Every text style is matched by family + style +
  weight. A missing font is asked for, never approximated.

In `faithful` mode the frame's visible copy, imagery, fonts, colors, spacing,
hierarchy, and section order are binding. Keep live text as real text — never
rasterize a headline to an image unless the source node is outlined or
vector-only. Do not call image-generation tools; every image comes from the
source. Do not add sections the design does not have (proof, FAQ, risk
reversal, a conversion brief): if the design has a conversion gap, say so in
your summary and let the user decide.

## Reading the design

The operations you need, by job:

- `figma_get_images` renders nodes to PNG/JPG/SVG and returns a URL per node.
  Use it to actually look at the design (PNG at `scale: 2`) rather than
  inferring appearance from the node tree, and later to export image and
  vector nodes.
- `figma_get_file_nodes` returns the node tree for specific ids: auto-layout
  properties, text with its `style` object, fills, strokes, effects, corner
  radii, and component instances. Prefer it over `figma_get_file`, batch every
  id you need into one comma-separated `ids` value, and fetch large frames
  section-by-section rather than dumping the whole file. Pass a small `depth`
  only for orientation.
- `figma_list_file_components` and `figma_list_file_styles` give the file's
  real component and token names — reuse them instead of inventing structure.

Figma's file-content endpoints are strictly rate-limited, hardest on
View/Collaborator seats, and a whole-file read on a large document will
exhaust that budget for little benefit. If a call comes back rate-limited and
the message says the seat's limit is monthly, do not retry: tell the user
their Figma seat limits API access and work from what you already fetched.

Writing to Figma — posting a file comment — is not on the public surface. Only
reads are exposed, so say so and offer a `start_agent_session` prompt if the
user asks for one.

## Lock the source before the first page write

Before writing anything substantive to the route:

1. **Render the exact target node** with `figma_get_images` at `scale: 2` and
   save the image locally so you can re-view it per section.
2. **Fetch the exact target node** with `figma_get_file_nodes`.
3. **Find the corresponding viewport.** When coverage is desktop and mobile,
   inspect the target's siblings (a shallow fetch of the parent) for an
   explicitly corresponding frame of the same page, such as `Home / Desktop`
   and `Home / Mobile`. If one exists, render and fetch it too. Do not
   substitute a similarly named but unrelated design.
4. **Confirm every node id and name** matches the requested page and viewport.
5. **Write a compact source manifest** and keep it beside your work: file key,
   node ids and names, frame widths, coverage, validation widths, visible copy,
   font inventory, colors, image and vector node ids, and local render paths.
   Validate at the source frame width for each represented viewport; when a
   viewport has no source frame, validate desktop at 1440px and mobile at
   390px.

The **font inventory** lists every distinct text style with its `fontFamily`,
`fontStyle`, `fontWeight`, and `fontPostScriptName` (when present) — read from
each text node's `style` object — plus an example node id and sample
characters. Record the full family + style + weight, never the family alone.
If the design uses a font the site does not have, **ask the user for the font
file** rather than approximating with a lookalike — a substituted typeface is
the most visible way a rebuild stops matching its source.

An empty `children` array on a container frame is usually a shallow-`depth`
artifact, not a true leaf. Re-fetch that node id without a restrictive depth
(or section-by-section) before treating it as structureless. Only rebuild from
the render alone when a full-depth fetch still returns nothing, or you are
rate-limited.

## Build from the locked source

Work one frame at a time, and build the page section by section under the
skeleton-first rule from **building-replo-pages** (first write is the minimal
skeleton; the route renders after every edit).

1. **The frame is the page spec.** Do not start from the template library, a
   registry section, or a section you remember; reproduce what the frame
   shows. Re-view the render before composing each section — build what you
   see, not what you remember.

2. **Translate auto-layout to flexbox** from the node data, never from the
   pixels: `layoutMode` → `flex-row` / `flex-col`; `itemSpacing` → `gap`;
   `padding*` → padding; `primaryAxisAlignItems` → `justify-*`;
   `counterAxisAlignItems` → `items-*`; `layoutWrap` → `flex-wrap`; sizing
   `FILL` → `flex-1` / `w-full`, `HUG` → `w-fit`, `FIXED` → an explicit size;
   `layoutGrow` → `grow`. Carry `cornerRadius`, strokes, `opacity`, fills
   (solid and gradient), and `effects` (drop and inner shadows, blurs) across.
   Absolute-positioned children inside auto-layout become `absolute` inside a
   `relative` parent.

3. **Reuse the source design system.** Component instances map to one React
   component per Figma component, named after it; style and variable names
   become your token names.

4. **Colors: faithful values, still tokens.** Read the exact values from fills
   (`r`/`g`/`b` on a 0–1 scale, with `opacity`) or from the file's styles.
   Then register them as CSS variables in `app/globals.css` and consume them
   through Tailwind theme classes, exactly as `references/styling-tokens.md`
   in **building-replo-pages** prescribes. When the Figma design *is* the
   site's design, update the site-wide tokens; when it is one page with its
   own palette, scope the variables under a wrapper class on that page's root
   so the rest of the site is untouched. The Definition of Done greps for raw
   hex in components — a hex that matches Figma is still a defect there.

5. **Load the exact font for every text style.** For each font-inventory
   entry, follow `references/fonts.md` in **building-replo-pages**: a web font
   through `next/font/google` with the exact `weight` and `style` arrays, a
   supplied file through `next/font/local` with one `src` per face. Never fake
   a style with a CSS `italic` or `font-bold` class or a nearby family. If the
   font is neither a web font nor supplied as a file, stop on that text: tell
   the user exactly which family, style, and weight is missing, ask for the
   file, and leave that text unfinished rather than shipping a lookalike.

6. **Persist every image as a Replo asset.** `figma_get_images` returns
   **temporary signed URLs** that stop resolving after about 30 days; a page
   referencing one looks correct today and serves broken images later. For
   each image or vector node, render it (PNG at `scale: 2` for raster, `svg`
   for icons and vectors) and, while the URL is still live, hand it to
   `upload_asset` as its `url` with a descriptive `name` and `altText`. It
   fetches the bytes and returns a permanently hosted asset — reference that
   hosted URL in `next/image`, never the Figma one. Run `find_assets` first
   for assets the project already holds (the logo, product photography) and
   reuse those. If a node cannot be exported individually, export and upload
   its parent; if that also fails, report the blocker instead of inventing a
   substitute.

7. **Real text from the node tree**, never OCR from the render. Copy is
   binding in `faithful` mode; fix only obvious gibberish, and keep the voice.

8. **Responsive.** With two frames, each is binding at its own width. With
   one, keep the source viewport exact and derive the other with natural
   reflow, readable type, usable navigation, and touch-safe controls, without
   changing content, hierarchy, visual language, or section order.

## When the design shows products

A Figma mockup usually contains placeholder product content. Wire the real
catalog instead of hardcoding what the design happens to show — read
**shopify** for the loader architecture and **product-display** for the rules
on titles, images, pricing, and availability. The design governs layout and
styling; the store governs the data.

## Completion gate

A route that returns 200 proves runtime health, not design fidelity. Before
reporting done:

1. The route returns 200 with no runtime or build errors and clean TypeScript.
2. Capture the page at every source frame width and compare each capture
   directly with its Figma render, section by section.
3. Capture and inspect every requested viewport — both desktop and mobile
   unless the user limited coverage, including the derived one.
4. Walk the source manifest: visible copy, every text style's font (family +
   style + weight, loaded for real or flagged as missing), primary colors,
   source assets, hierarchy, and section order are all represented.
5. In `faithful` mode: no image-generation tools were called, no
   destination-brand substitutions were introduced, no section was added or
   removed.
6. Run the **building-replo-pages** Definition of Done and its visual audit.
7. Fix material discrepancies before reporting. The final summary includes the
   route, the original Figma URL and target node, per-viewport fidelity
   evidence, and any font or asset still outstanding.

Publish only when the user explicitly asks — see the **publish** skill.

## Correcting a page built from Figma earlier

"The fonts are wrong", "the spacing is off", "the hero image is missing" on a
page built from a Figma design: re-lock the source (render + node data for the
affected frame), diff the page against it, and fix from the node data. Do not
tune by eye. The most common defects are a faux italic or bold standing in for
a real face, a Figma image URL that expired, and a section rebuilt from memory
instead of from the frame.

## Related Skills

- **building-replo-pages** — the operating manual for the site repo: clone, skeleton-first, tokens, `next/image`, fonts, and the Definition of Done that every Figma build must also pass.
- **publish** — the only way to deploy; a rebuilt page is not live until the user asks for it to be.
- **replo-branding** — when the Figma file is the source of the *brand* (colors, fonts, logo) rather than one page; a brand kit is a different job from a page rebuild.
- **shopify** and **product-display** — real catalog data behind the product content a mockup shows.
