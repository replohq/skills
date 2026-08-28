# Fonts

Single source of truth for how fonts are loaded, matched, and applied on Replo Next.js sites. Anything that touches fonts — building or styling a page, applying branding, or rebuilding a design from Figma or a URL — follows this file.

## How a Replo site loads fonts

Fonts flow through one seam, the same way colors do (see `styling-tokens.md`):

1. `app/layout.tsx` loads each font with **`next/font`** and assigns it a CSS variable (e.g. `variable: "--font-sans"`).
2. The variable class is placed on `<html>`/`<body>` so the variable is in scope.
3. `app/globals.css` maps the font token to that variable (`--font-sans: var(--font-geist-sans)`), and components consume the token via Tailwind (`font-sans`) or `var(--font-*)` — never a raw `font-family`.

Because the token layer routes `font-sans` to the variable, swapping the `next/font` import in `layout.tsx` re-fonts the whole site from one place. Load every font through `next/font`; do not hand-write `@font-face` blocks.

## A font is a family **and** a style **and** a weight

"Inter" is a family. "Inter / Italic / 600", or a named style like "ITC Garamond Std / Light Condensed Italic" (PostScript `ITCGaramondStd-LtCondIta`), is the specific thing you must load.

- Request the exact weights and styles you need — never load only the family and let the browser fake the rest.
- **Never approximate a style.** A CSS `italic` or `font-bold` class, or snapping to a nearby family/weight, produces browser-synthesized faux italic/bold — a fidelity defect for brand and design work.

## Loading a web (Google) font

```tsx
// app/layout.tsx
import { Inter } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  weight: ["400", "600"],
  style: ["normal", "italic"],
  variable: "--font-sans",
});
// add `inter.variable` to the <html>/<body> className
```

Then map the token in `globals.css` (`--font-sans: var(--font-sans)` or the role that should use it).

## Loading a supplied / self-hosted font file

Use `next/font/local` with the file(s) committed under the project (e.g. `app/fonts/`). One `src` entry per face, with `weight` and `style` set to match each file:

```tsx
// app/layout.tsx
import localFont from "next/font/local";

const garamond = localFont({
  src: [
    {
      path: "./fonts/ITCGaramondStd-LtCondIta.otf",
      weight: "300",
      style: "italic",
    },
  ],
  variable: "--font-display",
});
```

A variable font can use a single `src` with a `weight` range. Apply the resulting `--font-*` variable to the specific text (e.g. a headline) rather than the whole site when it is a one-off.

## Brand fonts vs one-off page fonts

- **Brand font** (the font a site uses everywhere): lives in brand context — `DESIGN.md` `tokens.fonts` (each entry has `family`, `delivery` [`google-fonts` | `cdn` | `uploaded` | `system`], `weights`, `styles`, and `faces[{ cssFamily, weight, style, localUrl, ... }]`). Apply it site-wide with the **apply-branding** skill (edits `layout.tsx` + `globals.css`). To add or swap a brand font, or to upload a self-hosted face, ask a Replo session — it writes the face's hosted URL into the brand kit; don't hand-author a local path.
- **One-off page font** (e.g. matching a single Figma headline): wire it directly on the site with `next/font` (google or local, per above) without touching brand context. If the font is becoming the brand's font going forward, ask a Replo session to update the brand kit instead.

## Matching a font from a source (Figma, a URL, a screenshot)

1. Read the exact family + style + weight from the source. For Figma, each TEXT node's `style` object carries `fontFamily`, `fontStyle`, `fontWeight`, and (often) `fontPostScriptName`.
2. Load that exact combination via `next/font` — `next/font/google` if it's a web font, otherwise `next/font/local` with a supplied file.
3. If the font is not a web font and you do not have the file, **do not approximate it.** Tell the user exactly which font (family, style, weight) is missing and ask for the file; that text is not finished until the real font is loaded.

## Verify

- The requested family actually renders (not a fallback). Confirm the `next/font` import and its `--font-*` variable appear in `layout.tsx`, the token in `globals.css` points at that variable, and the target text resolves to it.
- No faux styling: no CSS `italic`/`font-bold` (or a nearby family/weight) standing in for a real style the design calls for.
- If you were asked to match a font and could not load the real one, say so explicitly rather than shipping a lookalike.
