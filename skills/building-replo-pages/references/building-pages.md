# Building and Editing Pages

Create, edit, extend, and redesign Replo Sites pages with registry source as quality material and the current context as the source of coherence. Read this when planning, composing, editing, extending, or redesigning a page or section.

The registry is your source of _section quality_ — it's why pages look intentional and idiosyncratic instead of like generic AI layouts. The current site, brand, page, brief, or explicit source URL is your source of _coherence_. Use actual registry source without erasing the thing the user is asking to continue, change, or reproduce.

The operating slogan is: **registry-sourced, context-adapted.** Discovery alone is not enough. For new major sections and major replacements, find a registry candidate, download/stage it when applicable, read the source, and adapt from that source. Do not use the registry as visual inspiration and then hand-roll the section from scratch.

## 0. Operating frame

Before writing, decide the task type and design source of truth. This choice controls how much freedom you have.

### Task type

- **New greenfield page:** decide a domain-specific visual lane (or ask the user for direction), establish semantic tokens in `app/globals.css` if the project has no real brand yet, then compose sourced sections adapted into that lane.
- **Edit existing page/section:** inspect the existing route and local components first. Preserve working structure, tone, tokens, and responsive behavior unless the user asked to change them.
- **Add related pages:** match the existing site's rhythm, tokens, navigation, CTA language, footer, and section density before inventing new patterns.
- **Apply or adjust branding:** use the **apply-branding** skill for official project brand application. Do not repaint page JSX when the token layer can carry the change.
- **Use a specific template the user names:** ask a session for that item by name, then adapt it as implementation material.
- **Replicate a URL:** ask a Replo session to replicate it — cloning a page from a URL is done in a session, not from this reference.

### Design source of truth

- **Existing site/page:** context-first. Registry sections should bend toward the existing design language.
- **Existing project brand:** brand-first. Do not invent a new identity; keep components token-first.
- **Chosen design direction:** direction-first. When the user has picked a visual direction or supplied a mockup, it governs layout, palette, typography roles, hero treatment, and rhythm; sourced sections serve the direction.
- **Greenfield brief:** brief-first. Offer the user a visual direction before building (a Replo session can generate direction mockups to choose from). Otherwise invent a light brand direction from the brief and make section choices serve it.
- **External brand/source URL:** branding-first unless the user explicitly asked to clone the page, in which case ask a Replo session to replicate it. A URL can be a brand/style reference without being a replication target.
- **Registry:** structure-first. A registry item supplies layout quality and interaction patterns; it does not override the project's brand, voice, or route intent.

If two unrelated projects would look interchangeable after swapping nouns and logos, you ignored context. If a follow-on page looks unrelated to the existing site, you ignored coherence. If an edit turns into a full redesign without the user asking for it, you overreached.

## 1. Strategy before sections

Don't start from a fixed section skeleton. Start from the page's job:

- **Page type** (landing, product, about, collection, …) and **audience**.
- **The one offer / value prop** a visitor must grasp in ~3 seconds.
- **The main objection** to overcome and the **primary action** you want.

On a commerce page this strategy _is_ the Conversion Brief you post before the first write (see the spine in `SKILL.md`): these bullets, plus the objection each section answers and the offer structure you chose.

Then choose sections that each answer a buying question. Every section must earn its place with a distinct conversion job — don't pad to hit a number. But don't starve the page either: match the section density of analogous registry page templates (real DTC home pages run ~8–14 sections), not a minimal 3–4 skeleton.

If context is sparse, infer reasonable defaults and proceed — page composition is autonomous; do not stop to ask.

For edits and related pages, strategy starts by reading what already exists. Keep the parts that are working, identify the specific conversion or clarity gap, and change the smallest surface that solves the user's request.

### Borrow structure from real page templates (do this before skeletoning a new page)

Don't invent the section count and order from the generic archetype when the registry already contains real pages that solved this exact problem. For a **new page**, derive the structure from real page templates first, then source the individual sections:

1. **Pull 1–3 analogous page templates.** Ask a session for page templates of the same page type and closest brand/vertical — phrase the request by the page's job *and* vertical ("DTC supplement landing page with dense social proof", not "landing page"), and ask for the 1–3 closest matches.
2. **Read their structure, not their content.** Read the returned page source and extract the real top-to-bottom **section ordering and count** — the sequence of section _jobs_ (hero → proof → showcase → benefits → …), not the brand-specific copy or imagery.
3. **Let that observed structure set your skeleton.** Use the count and ordering from the analogs as the plan. Match their density rather than defaulting to a minimal page; deviate only where this page's strategy genuinely argues for it.
4. **Then source each section.** With the skeleton fixed, request and adapt individual sections the same way. You borrowed _structure_ from the page templates and _section quality_ from the section templates.

Fall back to the generic archetype below only when no page template is a usable structural analog for this page's type and vertical.

### File placement and component boundaries

- Keep `page.tsx` and `layout.tsx` as Server Components. If a section needs state, effects, browser APIs, or event handlers, extract the smallest interactive leaf into a co-located client component beside the route (for example `app/about/HeroSection.tsx`) and import it from the server page.
- Do not split static sections into extra files just for organization; a simple static page can stay in one `page.tsx`.
- Page-specific components belong under the route they serve. Put files in `components/` only when they are meant to be reusable across routes: that directory is surfaced to users as Site Components in the editor, and its top-level files must follow the Site Component contract in `references/site-components.md` (one standalone file per component, no imports of other site files).
- Never author an import that reaches into another route's folder (for example `app/pricing/page.tsx` importing from `app/about/`). Users can delete or duplicate pages directly from the editor with no agent in the loop, so another route's folder can change or empty out at any time. If two routes need the same component, promote it to `components/` per `references/site-components.md` and import it from both. If you encounter an existing cross-route import while editing those files (page duplication can create them), promote the site component then.
- A route folder that has no `page.tsx` belongs to a page the user deleted (deletion removes only the entry file). Treat its remaining files as dead: never import from them or build new work on them. You may remove such a folder while working nearby, but only after verifying nothing imports from it (grep for the path first).

### Building from a chosen design direction

When the user picks a mockup, **the mockup is the page spec — build exactly what it shows.** No reinterpretation: same section sequence, same layout within each section, same palette and typography roles, same button shapes, same imagery subjects, copy as shown where legible (fix obvious gibberish, keep the voice). Nothing from the site's prior design system survives unless the mockup shows it.

- Download the mockup before the skeleton; set `app/globals.css` tokens and font variables to match the mockup before composing any section — the exemplar description and the project brand kit carry the exact hexes.
- **Re-view the mockup image before composing each section** — build what you see, not what you remember.
- Reproduce the mockup directly — write each section to match what you see. A registry section is optional reference, not the starting point: use one only when it *already* matches that section of the mockup, then bend it to the mockup; otherwise build the section from scratch. The **Registry-only** rule does not apply to a chosen-mockup build — the mockup is the spec, not the registry.
- **Front-load imagery.** Kick off image generation for the mockup's key subjects (hero, product) at the *start*, in parallel with the build, so real images are ready as sections land — source per `references/images-and-assets.md` (real assets first). No placeholder survives to the finished page.
- **Done is a convergence loop, not one pass:** screenshot the rendered route, compare to the mockup section by section, fix every divergence, and re-check — keep looping until the built page reads as the same page. Never finish with a placeholder image or an obvious mismatch.

### Domain visual lane

For greenfield pages, a user-picked direction supplies this lane (see "Building from a chosen design direction" above). Use the text-only derivation that follows when no direction exists.

Before choosing sections, translate the business description into three visible commitments:

- **World:** the environment, objects, or workflow the buyer recognizes (dock appointment board, site-readiness checklist, protocol timeline, inventory map, dispatch queue).
- **Palette and surface:** a small visual system that follows the domain cues instead of defaulting to black/purple SaaS. Coastal logistics might use deep water, fog, chart lines, manifests, and dock-status surfaces; regulated clinical ops might use white/ink, calm green/blue accents, audit trails, checklists, and protocol cards.
- **Above-fold proof:** one artifact in the first 1200-1600px that proves the product exists — dashboard mock, workflow diagram, compliance/protocol panel, site-readiness checklist, operational queue, or dispatch/inventory board. A nav + centered headline + CTA is not enough, and a stat row by itself is not enough.

When the project has no meaningful brand yet, set or revise `app/globals.css` semantic token values before building the page. When the project already has a real brand, use the brand; do not invent a second one because the current page request is new.

If two unrelated businesses would look interchangeable after swapping the logo and nouns, the page failed this gate. Fix the visual lane before continuing.

### Archetype starting points (fallback when no page template fits)

Use these only when step 1 above surfaced no usable page-template analog. A real registry page template's observed structure always beats these generic defaults.

- **Landing/home:** hook hero → trust/social proof → product or feature showcase → benefits/how-it-works → testimonials → collection/CTA path → final CTA → footer.
- **Product:** hero with product gallery → details (price, variants, add-to-cart) → benefits → reviews → related products → FAQ → footer.
- **About:** brand-story hero → mission/values → team/founders → timeline/milestones → press → CTA → footer.

Take a different order or fewer sections whenever the strategy or the available templates argue for it.

### Visual rhythm

Vary section heights and surface roles (`background` / `muted` / `card` /
accent bands), generous vertical padding, contained width (`max-w-7xl mx-auto`)
alternating with full-bleed. Rhythm means **contrast within the page's
light/dark lane** — not importing the opposite mode. Never drop a light-mode
section onto a dark page (or the reverse) to "break up" the page; that is how
sections become unreadable. If you want a banded section, keep both surface and
foreground paired for that band (see `references/styling-tokens.md`).

## 2. Skeleton first, render progressively

The preview pane is the user's only window into the build. A page that materializes as one big write at the end of the turn reads as "nothing is happening" for the entire build. Get the route rendering early and keep it rendering after every edit, so the user watches the page assemble instead of staring at a 404.

### The skeleton contract

For a new page, immediately after strategy (and greenfield tokens in `app/globals.css`, since the skeleton should render in the right lane), write `app/{route}/page.tsx` as a valid Server Component that renders standalone. On a localized site (an `app/[lang]/` directory exists), the page goes under `app/[lang]/{route}/page.tsx` instead — a page outside `[lang]` is unreachable behind the locale middleware; see `references/site-i18n.md`.

- **Real top-of-page content:** the page's actual H1/subhead — and the primary CTA if you already know it — from your strategy. Not lorem ipsum, not "coming soon".
- **One quiet placeholder block per planned section:** a token-surface block (`bg-muted`, an on-scale `min-h-*`, `aria-hidden`) with no copy. Placeholders are scaffolding; every one of them must be replaced or removed before you declare done.
- **Zero speculative imports.** The skeleton may import only files that already exist (the UI library, `next/image`, …). Importing a component you haven't written yet is the classic way to put the route into a broken state — never do it, in the skeleton or any later edit.

Confirm the route actually renders — request it on the local dev server and check for a 200 with no build error — then share the local URL so the user can watch the build. Revealing a broken route is worse than revealing late; render first, then reveal.

This sequence is not optional and does not scale with page size. It applies to every new page, including a small page you could comfortably write in a single shot: skeleton write → render check → tab open → then content. "The page is quick, I'll just write it all and show it at the end" is exactly the behavior this contract exists to prevent — the user cannot tell a fast hidden build from a stalled one.

For edits to an existing page there is no skeleton step — the page already renders. Point the user at the route before your first edit instead.

### Keep it rendering

Build one section at a time and land each one as it's completed:

1. Build the section: for an extracted component, write the file(s) and confirm they compile (clean LSP) before touching the route; for a simple static section, compose the JSX directly.
2. Edit `page.tsx`: replace that section's placeholder with the real section (adding the import if there is one).
3. Run the per-section exit gate, then move to the next section.

**One placeholder per edit.** This loop carries the same force as the skeleton contract; it is not a code-organization suggestion. Each `page.tsx` edit replaces exactly one placeholder, including on a single-file page where you compose every section's JSX directly. "I have all the sections ready, one edit is cleaner" is exactly the behavior this rule exists to prevent — batching finished sections into one big write turns the skeleton back into a wall the user stares at for the rest of the build.

**Never let asset work gate a landing.** Asset searches, downloads, and generation run in parallel with this loop, not ahead of it (see `references/images-and-assets.md`). While an asset job is in flight, keep landing sections: use `https://placehold.co/{w}x{h}` or land an image-light section first, then swap in the real `src` when the job completes. Front-loading the whole asset batch before writing any section serializes the two pipelines and recreates the dead preview.

After every `page.tsx` edit the page must still compile and render. Never leave the route referencing a file that doesn't exist, and never park broken JSX in it "to fix later" — the user is watching the preview the whole time.

## 3. Sourcing real source

Replo maintains a library of designed page and section templates. It is served
by the Replo agent, so you reach it by prompting a session — ask for the real
source, not a description, then pull the result into your clone.

- **Phrase requests by job and shape, not keywords.** "hero with operational
  dashboard proof and two CTAs" beats "hero". On commerce pages, phrase the
  conversion job _and_ the vertical: "supplement product page buy box with
  subscribe-and-save and ingredient benefits".
- **Ask for a primary and a backup** so you have something to choose between.
- **Read the source before adapting it** — the quality lives in the responsive
  intent, accessibility, and interaction wiring, not in the copy.

## 4. Structure over improvisation

Every new major section and major replacement should start from proven
structure — a direct fit or a structural analog — rather than generic JSX
written from memory. That is the whole reason the library exists: pages built
this way look intentional and idiosyncratic; pages improvised from memory look
like templates.

When nothing fits, stretch the closest analog before writing from scratch, and
never drop a required commerce section (price, shipping/returns, trust, spec
table). When you do write a section directly, hold it to the same bar: a
distinct conversion job, real copy, and a clean pass through the slop detector
below.

**Exception — a chosen design mockup:** the mockup is the spec, so reproduce it
directly and build sections to match it.

**Exception — the user's own existing markup** (promoting it into a Site
Component, or repeating it on another page): their markup is the source. See
`site-components.md`.

## 5. Adapting a template

Adapting means starting from the staged, lifted, or fetched JSX and substituting context-specific values — not reading it for inspiration and rewriting your own version (that reliably produces generic layouts).

**Preserve the quality-bearing contract:**

- Responsive intent — keep distinct mobile/desktop variants (`md:`, `lg:`, `hidden`/`block`); don't collapse them to "simplify".
- Accessibility structure and any interaction wiring (carousels, scroll-snap, accordions, hover/animation).
- Media composition — aspect-ratio containers, `objectFit`/`objectPosition`, overlay structure.
- Non-obvious density/rhythm decisions that make the section feel designed.
- Component decomposition and small repeated structures when they carry the design rhythm.

**Normalize freely (this is the point of adapting):**

- Color → semantic tokens (see `references/styling-tokens.md`). Convert arbitrary spacing to the Tailwind scale — `py-[40px]` _is_ `py-10`; converting it is correct, not a violation.
- **Page theme lane:** detect whether the destination page is visually dark or light, then normalize the registry section into that lane. A light-authored registry section on a dark page (or the reverse) without this pass is a defect — often unreadable, always jarring. Prefer token surfaces that inherit the site; if you intentionally band a contrasting section, pair every surface with its matching foreground on that section.
- Container width, typography roles, CTA treatment, copy, image `src`/`alt`, `href`.
- Irrelevant substructure that doesn't serve this page.
- Existing-site continuity: if the page already has a clear CTA shape, heading scale, card radius, or navigation pattern, normalize the registry section into that system rather than importing a second visual system.
- Source-brand artifacts: remove source logos, product names, testimonial names, press marks, pricing, routes, tracking IDs, and assets that do not belong to the user's site.

**Adaptation pass order:**

1. Make it run in the local scaffold without breaking imports/deps.
2. Convert styling to semantic tokens, match the page's light/dark lane, and align radius/type/CTA roles.
3. Replace copy and information architecture with the user's actual page story.
4. Replace imagery/assets and `alt` text.
5. Check the section still has the original registry item's layout strength after adaptation — and that every text/icon on every surface in the section is legible on the live page.

**Traceability, not audit theater:** a missing comment is not a defect. Keep a single ledger near the page rather than a comment on every block:

```tsx
// registrySources: hero → ridge-wallet-home-hero; reviews → composed(quote-grid + avatar-row); guarantee → single-column-cta (stretched analog)
```

Inline `// registry: <slug>` comments are worth it only on heavily-copied complex interactive sections.

## 6. Images

Real assets beat placeholders. Order of preference: user assets (`find_assets` — omit the query to list all) → images generated by a Replo session → `https://placehold.co/{w}x{h}`. Never ship a broken/empty `src`. Every image gets a descriptive `alt`. Full strategy in `references/images-and-assets.md`.

## 7. Cohesion

Pick or inherit **one visual lane** up front (density, image treatment, rhythm, CTA style, tone) and keep sections in it. Cohesion is not _sameness_ — a hero CTA can legitimately differ from a footer link. Define small **role systems** instead of one-of-everything: a heading role + a body role; one primary CTA role (`bg-primary text-primary-foreground`) reserved for the page's single primary action, with every other button on an outline/secondary/ghost variant. Write all copy in one consistent voice.

For follow-on pages, inherit before inventing: use the existing site's navigation, footer, tokens, core CTA treatment, and page rhythm unless the user explicitly asked for a divergent campaign concept.

## 8. Slop detector

These are normal UI primitives, not banned shapes — but if you're about to emit one as bespoke generic JSX, stop and either compose from the registry or make a deliberate non-generic choice:

- Centered-gradient hero with no product/visual proof above the fold
- Generic black/purple SaaS shell for a domain that asked for another world
- Giant centered text-only first viewport with no operational artifact, proof strip, or product/workflow visual
- A stat row pretending to be product proof when the prompt asks for workflow, compliance, clinical, logistics, or operational detail
- A lone row of 3 identical centered icon cards
- `max-w-7xl` on literally everything; uniform `py-8` everywhere
- Card grid of image+heading+body+link with no registry source
- Alternating image/text rows, numbered step lists, quote+avatar blocks, logo strips, stat rows, single-column accordions emitted as generic JSX
- Filler copy ("Empower your workflow"); fabricated logos, press, awards, certifications, or review counts — a greenfield brand still gets a swappable placeholder review block, per the greenfield proof rule in the `cro-universal-rules` guidance
- New pages that ignore the existing site's visual language
- Edits that silently redesign more than the requested surface

## 9. Per-section exit gate

Run this as you finish each section (not as a deferred final pass):

- Does it have a distinct conversion job, or is it padding?
- Did it come from a staged/lifted registry item or a stretched analog? (If neither, it's a defect — redo from a fetched analog or drop the section. Original JSX is not allowed.)
- If this is an edit or follow-on page, does it still feel native to the existing site?
- Are responsive variants and a11y/interaction wiring intact?
- Color via tokens only; spacing on the scale; no arbitrary values?
- Does the section match the page's light/dark lane (or use an intentional band with paired surface+foreground)? Is every text/icon legible on its surface?
- Real imagery with `alt`; no broken `src`?
- Does its shape match the slop catalog? If so, fix before moving on.

Page-level, before done: consistent palette/typography/CTA roles, smooth rhythm, no leftover template brand colors or placeholder copy, clean LSP, and the root Definition of Done gates pass. Give the page accurate `metadata` (a specific `title` and `description`) — the site's `/llms.txt` is regenerated from page metadata on publish, so good metadata is how a page shows up well there (see the **site-metadata-conventions** skill); never hand-edit `llms.txt`.
