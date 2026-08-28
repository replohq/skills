# Site Components

Read this when creating, editing, renaming, or deleting anything at the top level of the site's `components/` directory, when promoting existing page markup into a reusable component ("use my same header on this page", "make this a component"), when the user mentions a Site Component in chat, or when the Site Components preview needs an update.

## What the user sees

Every top-level `.tsx` file directly in the selected site's `components/` directory is surfaced — by file name — as a Site Component in Site Builder's Site Components view. There the user previews the component in isolation, customizes its props in a generated props panel, and can mention it in chat (the mention hands you the file's path). Subdirectories are not surfaced.

That surfacing drives every rule here: the top level of `components/` is a user-facing product surface, not a code-organization tool. A stray helper file becomes a broken entry in the user's component library; a required prop with no default becomes a broken preview; an untyped prop bag becomes an empty props panel.

## The contract — every top-level file in `components/`

1. **One component per file, one standalone file per component.** The file default-exports the component and contains everything it needs: subcomponents, helpers, constants, and types are inlined in the same file. Never split a Site Component across sibling files and never add a top-level helper or util `.tsx` — every top-level file is surfaced as a component, so a helper shows up in the user's library as a broken one.
2. **No imports of other site files.** Allowed imports: npm packages (`react`, `next/*`, `lucide-react`, …), the UI kit `@/components/ui/*`, `@/lib/utils` (`cn`), and `@replohq/sdk` (types and runtime hooks alike — a header's `useCart` is fine; the preview renders inside the site's root layout, so provider-backed hooks work there). Not allowed: routes (`@/app/...`), data files, other Site Components, or anything else in the site. If two Site Components want to share markup, inline a copy in each — self-containment is what keeps a component previewable on its own and safe against edits and deletions elsewhere in the site.
3. **It must render bare.** The preview mounts the default export with no props at all. Make every prop optional with a real default (destructuring defaults), and make the zero-prop render look finished — the defaults are the component as the user first sees it. When promoting from a page, the source page's current values become the defaults.
4. **Content arrives via props, not fetches.** A Site Component never fetches its own content; pages do the loading. A component that renders a product takes `product: FullProduct` (type imported from `@replohq/sdk/loaders/loader-schemas`) — the props panel then gives the user a real product picker and the preview loads real product data. Never flatten a product into scalar props (title, price, imageUrl); one `FullProduct` prop beats five strings. Live site state read through SDK hooks (cart contents, consent) is not "content" — those are allowed per rule 2.
5. **Tokens only, as everywhere.** Semantic-token styling is what lets one component sit on any page; a Site Component with raw color is broken on arrival (`references/styling-tokens.md`).
6. **Server-first as usual.** Add `"use client"` only when the component needs state, effects, or handlers, exactly like any other component. Prop edits in the preview apply in place for client components and via a refresh-style re-render for server components; both are correct, so never add `"use client"` just for the workbench.

Names are shown to the user split into words and sentence-cased (`SiteHeader` → "Site header", `ctaLabel` → "Cta label"). Name files in PascalCase after what the user calls the thing (`SiteHeader.tsx`, `TestimonialCard.tsx`, `AnnouncementBar.tsx`) and name props after the knob a merchant would recognize, not implementation jargon.

## Props are the customization surface

The props panel is generated from the component's TypeScript prop types, JSDoc, and defaults. Which props you expose decides what the user can customize without you. Only serializable props surface:

| Prop type you declare                          | Control the user gets      |
| ---------------------------------------------- | -------------------------- |
| `string`                                       | text input                 |
| `number`                                       | number input               |
| `boolean`                                      | toggle                     |
| string-literal union (`"left" \| "center"`)    | select with those options  |
| `Date` / documented ISO date string            | date-time picker           |
| `FullProduct`                                  | product picker             |
| plain object or array                          | JSON editor                |

Callbacks, refs, `ReactNode`, `className`, `style`, and anything non-serializable never surface — don't build the customization story on them. Concretely:

- Declare an explicit props type on the component and give non-obvious props a one-line JSDoc; both feed the panel.
- Prefer a string-literal union over a boolean for anything with more than two meaningful states (`variant: "light" | "dark"`, `align: "left" | "center"`).
- Model repeated content (nav links, testimonials, logos) as an array-of-objects prop with a typed element — it surfaces as editable JSON. Keep such values small; the prop channel caps at 16KB.
- Expose the knobs a merchant actually wants — headline, CTA label and href, image URL, alignment, product — not internal tuning values.

## What does not go at the top level

- Page-specific sections stay in their route's folder (`references/building-pages.md`, file placement). `components/` is only for cross-page reuse.
- Internal shared code that must not appear in the user's library lives in subdirectories — `components/ui/` (the UI kit) and `components/consent/` already work this way.
- `app/replo-preview/` routes are editor-generated preview plumbing: Git-excluded, dev-only, never shipped. Never author, edit, commit, or clean up files there.

## Promoting page markup into a Site Component

These asks all mean "promote": "use my same header on this page", "make this header a component", "save this section to my components", "reuse this on the pricing page", "make my footer consistent everywhere". Promotion is an extract-refactor, never a redesign. The Registry-only rule does not apply — the user's own markup is the source, and reusing it is the point. Promotion and component edits are surgical changes, not page composition: the commerce planning path and Conversion Brief do not fire, while the standard gates still apply to every file you touch.

1. **Locate the canonical source** — the page or route-local component the user pointed at. If near-identical copies exist on several pages, the referenced one is canonical. If the "same" element already lives in `app/layout.tsx`, it is already on every page — the real request is a drifted per-page copy or a page bypassing the layout; find and fix that divergence instead of promoting a second copy.
2. **Extract into `components/Name.tsx`** meeting the contract above: standalone, default export, inlined helpers, tokens. When flattening from a subdirectory (`components/home/Header.tsx`), pick a distinct top-level name that still reads well beside its siblings (`SiteHeader.tsx`).
3. **Parameterize real per-page differences as props**, with the canonical page's current values as defaults. Don't invent knobs nobody asked for; a first promotion with zero or two props is normal.
4. **Replace the source usage** with an import of the new component so one implementation exists, not a copy. The source page must render identically after the swap — this step has no visual delta.
5. **Import it at the requested target page(s)**, passing props only where the target intentionally differs.
6. **Sweep the stragglers.** If the user asked for consistency ("same header everywhere") and other pages carry drifted copies, swap them too and note the unification in your summary. If they asked about one page, leave the other copies alone but mention they exist.
7. **Verify** the source page is unchanged and every touched route still renders; the standard gates apply.

Placement for "on every page": import the component on each page. Move it into `app/layout.tsx` only when the user explicitly wants it on every current and future page (a true global header or footer) — and never disturb the runtime layer there (`references/canopy-and-environment.md`).

## Editing an existing Site Component

A Site Component lives at `components/Name.tsx`. Before editing, grep the site for imports of it — the edit lands on every page that uses the component, and your summary must say which pages changed. If the user wants a change on one page only ("make the header transparent on the landing page"), don't fork the file and don't change the shared default: add a prop and set it at that one usage. A page-scoped change intentionally leaves the shared default — and therefore the bare workbench preview — looking unchanged, so tell the user which page shows the change rather than letting them hunt for it in the preview.

Prop values the user dials in the Site Components view are preview state, not site code; nothing persists until you change the defaults or the page usages. If the user asks to make their previewed settings permanent, get the values from their message or ask which ones they chose — there is no channel that hands you their workbench state.

## Preview needs an update


## Revealing your work

After creating or promoting a component, tell the user it is available in Site Builder so they can preview and customize it.
