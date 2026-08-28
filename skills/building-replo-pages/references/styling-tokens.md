# Styling: Token-First

Read this before touching any color, font, or radius. The contract: color, font, and radius live **once** in the semantic token layer in `app/globals.css` (`:root`, `.dark`, `@theme inline`). Pages **consume** tokens; they never define their own. That single seam is what lets a brand be applied — or dark mode flipped — by editing one file instead of crawling every page.

## The one rule that matters

**No raw colors of any kind in components.** This is stronger than "no hex":

- No hex / `rgb()` / `hsl()` / `oklch()` literals.
- No arbitrary Tailwind color values: `bg-[#0a0a0a]`, `text-[#fff]`.
- No `style={{ color: "#000" }}`.
- **No built-in Tailwind palette classes** — `bg-slate-800`, `text-blue-500`, `border-gray-200` are equally un-themeable and slip straight past a hex check. They are the most common token bypass.

Source every color from a semantic role: a Tailwind class (`bg-primary`, `text-muted-foreground`, `border-border`) or, inside an inline `style`, a CSS var (`backgroundColor: "var(--primary)"`).

## globals.css is authoritative — read it

Don't trust a hardcoded token list in any doc; token sets drift and a confidently-wrong list is worse than none. Read `app/globals.css` for the real set. The semantic roles it defines are, broadly:

- **Surfaces:** `background`, `card`, `popover`, `muted`, `accent` — each with a paired `*-foreground`.
- **Actions:** `primary`, `secondary` (+ `*-foreground`), `destructive`.
- **Structure:** `border`, `input`, `ring`.
- **Shape:** `radius` (use `rounded-md`/`rounded-lg`, not `rounded-[10px]`). **Fonts:** `font-sans` — for how fonts are loaded, matched, and applied (`next/font`, brand vs one-off), see [fonts.md](fonts.md).

Prefer these semantic roles over the raw `--color-brand-*` palette vars.

## Pair every surface with its foreground

Whenever you set a background, set the matching foreground from the _same_ family: `bg-background`→`text-foreground`, `bg-card`→`text-card-foreground`, `bg-primary`→`text-primary-foreground`, `bg-muted`→`text-muted-foreground`, etc. Never set a surface without its paired foreground — that pairing is what keeps text legible when the theme flips light↔dark. If you change a surface token on an element, change its foreground to match; never mix roles.

## Match the page's light/dark lane when adding sections

Before inserting or adapting a section onto an existing page, decide whether that page is visually **dark** or **light** (dominant section surfaces and `--background` in `app/globals.css`). Registry sections are often light-authored; the default Replo starter ships neutral light-mode. Landing a light-mode section on a dark page (or the reverse) without a theme pass is a defect — the classic failure is unreadable text when a section keeps a light surface but inherits the dark page's light foreground (or vice versa).

Rules:

1. **Prefer inheriting the page lane** via semantic tokens (`bg-background`/`text-foreground`, `bg-card`/`text-card-foreground`, …) so the section skins with the site.
2. **Do not use opposite-mode banding as decoration.** Rhythm comes from muted/card/accent roles inside the same lane, not from importing a light template onto a dark page.
3. **If you intentionally band** a contrasting section, set surface **and** every text/icon foreground for that band on the section itself — never change only `background` and inherit the page's text color.
4. **Legibility check before done:** every text/icon on every surface in the new section must be readable on the live page.

## Avoid default-token blandness

A page styled only with `bg-background`/`text-foreground` is technically themeable but lifeless. Before styling, make a tiny theme plan in your head: base surface, one elevated surface (`card`/`muted`), the primary CTA, an accent, border, and muted text. Use that handful of roles deliberately.

## When the semantic set genuinely lacks a color

If a section needs a color the semantic roles don't cover (e.g. a second accent), **add a new CSS variable to both `:root` and `.dark` in `app/globals.css`** and reference it as `var(--your-token)`. Do not inline a raw hex — a color that lives outside `globals.css` can't be re-skinned.

## Payoff

Because all color lives in `globals.css`, applying a brand (the **apply-branding** skill) rewrites only that one file plus the font and logo — it never has to touch your page components. Run the color gates in the root `SKILL.md` Definition of Done to confirm nothing leaked.
