# Image sizing in Safari

Read this when an image renders at the wrong size in Safari — too large, wrong
aspect ratio, or collapsed — especially when a height is set but not a width, or
the image is inside a `<picture>`. For images collapsing inside a flex/grid
parent specifically, also see `flex-grid-and-height.md`.

- **Symptom:** images render at the wrong size — too large, wrong aspect ratio, or collapsed — only in Safari, especially when a height is set but not a width.
- **Root cause:** several distinct WebKit bugs around `<img>` inside `<picture>`, percentage heights, and intrinsic sizing. Notably, Safari uses the **non-rendered (intrinsic)** image size to compute a `<picture>` width when the image has an implicit height.
- **Fixes (all load-bearing — do not prune as "redundant"):**
  - The `<img>` gets `min-width: 100%; max-width: 100%; min-height: 100%; max-height: 100%; height: 100%`. All five are required for Safari to size images correctly in different configurations. **Do not add `width: 100%`** — in Safari that breaks height-only layouts (it renders the image oversized instead of preserving the intrinsic aspect ratio).
  - When the parent has a height and the image has an implicit height, force `height: 100%` so the `<picture>` width is computed correctly (affects non-Chrome browsers).
- **Next.js/Tailwind:** the cleanest fix on a Next.js page is to give images an intrinsic box via `aspect-ratio` or explicit `next/image` dimensions plus `object-cover`/`object-contain`, which avoids the whole class. When you must fill a parent, mirror the min/max/height stack and never add `width: 100%` to a height-only image.

## Common mistake

- **Adding `width: 100%` to an image to "fix" its size.** In Safari, setting `width: 100%` on an image that has a height but no width breaks the intrinsic aspect ratio and renders it oversized. Prefer `aspect-ratio` or the min/max/height stack above.
- **Removing a "redundant" property from the min/max/height stack.** The redundancy is deliberate and covered by visual-regression tests in the runtime; each property is load-bearing in a specific Safari configuration.
