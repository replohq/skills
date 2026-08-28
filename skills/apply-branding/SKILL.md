---
name: apply-branding
title: Apply a Brand to a Site
summary: Restyle a site's colors, fonts, and logo from its design tokens.
description: "Use when restyling a Replo site to match a brand — token-first edits to globals.css and the layout font and logo, with hardcoded-color audits and contrast checks. Never repaint page components."
tools: list_projects, list_sites, create_api_key, publish_site
---

# Applying a Brand to a Replo Site

Re-skin a Replo site with a brand. This is almost entirely a **token edit**: Replo sites are built token-first, so every color/font/radius flows from the semantic token layer in `app/globals.css`. Applying a brand means rewriting those **token values** (plus the font and logo) — **not** repainting page components.

You still use judgment to map the brand palette onto the semantic roles and to choose readable foregrounds, but you should not be hand-editing section JSX. If you find yourself opening `app/page.tsx` and swapping hex codes, stop — that means either the site wasn't built token-first or you're working the wrong layer.

## Working on a Replo site from your own machine

A Replo site is a Next.js repo you can clone and edit directly:

1. Resolve the site: `list_projects` → `list_sites` (use the default site unless the user names one). Each site returns a `clone_url`.
2. Mint a key with `create_api_key` (include `repo.write` when you will push) and clone per the [Replo Git docs](/git/get-started).
3. `pnpm install && pnpm dev` gives a local preview. The Replo agent is a second writer of the same repo — pull before editing and push when done; never force-push.
4. Publish with the `publish_site` tool only when the user explicitly asks.

## Getting the brand

The project's brand kit lives in Brand Studio, outside the site repo. There is no brand tool on the public MCP surface, and none is needed — a Replo session reads and writes Brand Studio for you.

- **From a Replo session (do this first).** Prompt `start_agent_session` with: "Report the project's primary brand: every color token with its hex, the font families and faces, and the logo URL. Do not change anything." The session reads Brand Studio and returns the values. Do not ask the user to retype colors Replo already holds. If the project has no brand yet, ask the session to create one first (see the Brand kits skill).
- **From the user** — only when they are deliberately applying a brand the project does not have, or overriding specific values.

For the logo, use a URL that actually resolves; if there is no usable logo, leave the current logo untouched and say so.

## Steps — edit the token layer, not the pages

Make essentially all changes in **three places**:

1. **`app/globals.css` — the design tokens.** Rewrite the token _values_ in both `:root` **and** `.dark`. Map the brand palette onto the semantic roles using judgment:
   - Pick the brand's dominant/action color for `--primary`, and set `--primary-foreground` to a color that is **readable on top of it** (don't just copy a brand color in — verify contrast).
   - Set neutral surfaces (`--background`, `--card`, `--popover`, `--muted`, `--accent`) and each paired `*-foreground` so every surface/text pair is legible.
   - Set `--secondary`/`--secondary-foreground`, `--border`, `--input`, `--ring`, and `--radius` to match the brand's feel.
   - Keep `.dark` legible too: derive a dark variant of the palette (don't leave `.dark` as the old default while `:root` becomes the new brand). Every `*-foreground` must contrast its surface in both modes.
   - If the brand needs a color the semantic set lacks, add a new var to **both** `:root` and `.dark` rather than hardcoding it in a page.
   - Do **not** blindly map the first brand color to `--primary` or assign every brand color to a token 1:1 — assign by role and contrast.
2. **`app/layout.tsx` — the font.** Swap the `next/font` import and the `--font-*` variable to the brand font, only if it loads cleanly and doesn't break layout. The token layer already routes `font-sans` to this var, so this one edit re-fonts the whole site. **After swapping, assert the brand font actually took:** the brand's font name must appear in `layout.tsx` and resolve through `--font-*`. If the brand names a font you can't load (no `next/font` entry, custom face missing), keep the existing font rather than leaving a half-applied/broken family — and say so in your output. Match the exact family, style, and weight; never fake a style with CSS `italic`/`font-bold` on a different face.
3. **The logo.** Update the logo asset/URL where the layout or header references it. **Gate this on the asset existing:** only point the markup at a logo URL/path that actually resolves. If there is no usable logo, leave the current logo untouched — never wire up a broken/empty image. Note the skip in your output.

**Do NOT** edit page components, copy, layout, spacing, or imagery to "make the brand fit." A token-first site re-skins entirely from steps 1–3. The one exception: if you discover a page that **hardcodes** a color (raw hex / arbitrary `bg-[#...]`) instead of consuming a token, that's a build-time defect — fix just that spot by pointing it at the right token (and its paired foreground), then move on. Never repaint the page.

4. **Verify** the site still builds and renders (`pnpm dev`, and `pnpm exec tsc --noEmit` clean). Then run the post-apply assertions below.

## Post-apply assertions (required — do not skip)

These exist because a token rewrite can be 100% correct and still be **invisible** if a page hardcodes its colors, or a brand can be applied with broken text contrast. Run all of them:

- **Hardcoded-color audit.** Grep the site's `app/` for raw color literals that bypass the token layer:

  ```bash
  # Run at the site repo root.
  rg -n "#[0-9A-Fa-f]{3,8}\b|rgba?\(|bg-\[#|text-\[#" app --glob '!**/globals.css'
  ```

  Every hit is a component painting around your tokens. Repoint each at the correct semantic token (and its paired foreground). `globals.css` is the only file allowed to hold literal color values. Re-run until the only hits are intentional non-brand values (e.g. a CDN icon URL).

- **Contrast.** For each surface/foreground pair you set (`background`/`foreground`, `card`/`card-foreground`, `primary`/`primary-foreground`, `secondary`, `muted`, `accent`), confirm the text pair clears a legible contrast bar in **both** `:root` and `.dark`. Fix any pair that doesn't before finishing.
- **Font.** Confirm the brand font (or your documented fallback) is what actually renders, not the scaffold default left behind.
- **Logo.** Confirm the logo in the header resolves to a real asset, or that you intentionally left the prior logo.

## Honesty

Only claim what you actually verified. If you could not load a visual preview of the result, **do not assert the page "looks great"** — state plainly that you applied the token/font/logo changes and which assertions you ran, and flag that a visual pass is still pending. Never describe design quality you didn't observe.

## Output

Tell the user what changed in plain language — name the files you touched (typically just `globals.css`, `layout.tsx`, and the logo). Keep it short and specific. Push your changes when done so the Replo agent and dashboard see them; publish only if the user asked.
