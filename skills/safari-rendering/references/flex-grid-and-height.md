# Flexbox, grid, and percentage height (the "flex-collapse" family)

Read this first when a child element renders squished, collapsed,
stretched, or the wrong height in Safari but looks fine in Chrome.

## The flex-collapse symptom

Inside a flex container, a child — most often an **image**, but also text blocks
or nested sections — renders **squished, collapsed to a sliver, stretched, or
the wrong height** in Safari. `flex-direction: column` layouts collapse in
height; `row` layouts collapse in width. On a page it reads as "the image
is tiny / the section is way too short / everything is smushed" only on the
phone or live page. WebKit is stricter than Blink on every cause below.

## Quick fixes for a Next.js/Tailwind page

```tsx
// A flex child that should fill/scroll and is collapsing → allow it to shrink.
// Tailwind: min-w-0 (row) / min-h-0 (column). This alone fixes most cases.
<div className="flex flex-col min-h-0"> … </div>

// A flex child (image, card) being squished by siblings → stop it shrinking.
<img className="shrink-0" … />

// A percentage-height element inside a flex/grid parent that collapses in Safari
// → give WebKit a definite containing block, or drop the percentage entirely.
<div className="flex items-center min-h-0 min-w-0">
  <img className="h-full w-full object-cover" … />
</div>

// Best of all for images: use aspect-ratio (or next/image width+height) so the
// box has an intrinsic size and never depends on a percentage-of-flex-parent height.
<img className="w-full aspect-[4/3] object-cover" … />
```

**Prefer `aspect-ratio` / explicit dimensions for images over percentage
heights** — it sidesteps the whole class of WebKit percentage-height-in-flex
bugs and is what `next/image` wants. Reserve `min-w-0`/`min-h-0`/`shrink-0` for
cases where a percentage/fill layout genuinely has to stay.

---

## a. `min-height`/`min-width: auto` prevents flex items from shrinking

- **Symptom:** a flex child that should fill available space instead overflows or pushes its siblings; a "fill available" section is too tall/wide only in Safari.
- **Root cause:** the default `min-width`/`min-height` of a flex item is `auto`, and a flex item is not allowed to shrink below its min size. With enough content the child expands past the space it's supposed to fill. WebKit enforces this strictly; Blink is more forgiving.
- **Fix:** set `min-width: 0` (row) or `min-height: 0` (column) on the flex child — but only when the parent is **not** `flex-wrap: wrap` (wrapping needs the natural min-size to wrap correctly).
- **Next.js/Tailwind:** `min-w-0` / `min-h-0` on the flex child. Guard against putting it on wrapping flex rows.

## b. Flex children shrinking when they shouldn't

- **Symptom:** an image or fixed-size child gets compressed by its siblings in a flex row/column, only in Safari.
- **Root cause:** default `flex-shrink: 1`. WebKit compresses content-sized children more readily.
- **Fix:** `flex-shrink: 0` on children that must keep their size.
- **Next.js/Tailwind:** `shrink-0`.

## c. Percentage heights need a _definite_ parent height

- **Symptom:** an `<img>` or child with `height: 100%` (or a percentage height) collapses to zero/tiny height inside a flex or grid parent — only in Safari.
- **Root cause:** the strict CSS spec reading (which WebKit follows) requires a **definite** height on the parent for a percentage height to resolve. WebKit does **not** treat a flex/grid track as a definite reference; Blink resolves it anyway.
- **Fix:** give WebKit a definite containing block. Wrap the element in a flex box and set `min-height: 0` (so the flex item establishes a resolvable height), or avoid percentage height entirely and use an explicit height / `aspect-ratio`.
- **Next.js/Tailwind:** prefer `aspect-[w/h]` or explicit `next/image` `width`/`height`. If a fill/percentage layout must stay, wrap in `flex items-center min-h-0 min-w-0` and use `h-full`. (Image-specific sizing bugs are in `image-sizing.md`.)

## d. Safari flex-vs-grid: grid child with `100%` height inside a flex column

- **Symptom:** a `display: grid` element inside a flex column renders with the wrong `100%` height in Safari.
- **Root cause:** the "infamous Safari flex vs grid issue" — percentage sizing on a grid child doesn't resolve against a flex-column parent.
- **Fix:** insert a flex wrapper around the grid that re-parents the percentage sizing (copy `flex-basis/grow/shrink/align-self`, force `100%`, and swap the parent's `justify-content`/`align-items` so auto-layout placement still works).
- **Next.js/Tailwind:** rarely needed on hand-written pages (avoid nesting a `%`-height grid directly in a flex column); if you hit it, add the intermediate flex wrapper.

## e. Absolute fill: `inset: 0` alone is unreliable

- **Symptom:** a `position: absolute; inset: 0` overlay/fill doesn't cover its container in Safari.
- **Root cause:** WebKit doesn't reliably honor the `inset` shorthand alone for absolute fill.
- **Fix:** also set the longhand `top/left/right/bottom: 0` alongside `inset: 0`.
- **Next.js/Tailwind:** `absolute inset-0 top-0 left-0 right-0 bottom-0` (Tailwind `inset-0` already expands to all four, so this is mostly a concern with hand-written inline styles).
