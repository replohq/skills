# Images and Assets

Ecommerce pages live or die by imagery. Read this when adding or replacing images. Order of preference: **real user assets → generated images → placeholder**. Never ship a broken or empty `src`, and every image gets a descriptive `alt`.

## 1. Check the user's assets first

The user's uploaded assets are the same library shown in the Assets mini-app. Use **`find_assets`**:

- **List everything** — call it with no `q` to see the full inventory (newest first). This is the reliable way to answer "what assets do I have"; don't conclude the library is empty from a search that returned nothing.
- **Search for a specific asset** — pass `q` to get results in relevance order:
  - `"product shot of supplements on white background"`
  - `"person using skincare product"`
  - `"logo"`, `"brand banner"`
- Optionally narrow either mode with `type` or `tags`. Returns up to 50 assets.

List first to know what exists, then search (or run several queries with different terms) to pull the right ones. Real product photos make a page dramatically more effective than any placeholder.

## 2. Generate when there's no suitable asset

If the user's actual request is the creative asset itself (ad creative, lifestyle shots, image variants, a product video) rather than imagery for a page you are building, ask a Replo session to produce it — that is a different job from page imagery.

For imagery within a build, ask a Replo session to generate images — generation is async:

- **Batch:** up to 10 images per call (hero + benefit graphics + lifestyle shots in parallel).
- **Reference images:** pass an existing asset for style-consistent output.
- **Flow:** the session starts a generation job and reports the hosted URL when it completes.
- **Bounded polling:** ~10–20 polls with short waits. If it fails/stalls, fall back to `placehold.co` or another asset and keep building.
- Start generation **early** (Phase 2) so it runs in parallel with writing code.
- **In-flight jobs never block a section landing.** This applies to asset downloads as much as generation: while a job runs, keep replacing placeholders — land the section with `placehold.co` and swap in the real `src` when the job completes. Don't sleep-and-poll with the page untouched.
- **Prompt quality:** describe the shot concretely — framing, surface, lighting, and what must stay out of frame.
- **Backgrounds:** for page imagery, prefer an intentional solid or contextual background. Do not ask for a generic "transparent background" for cutout-style compositions—the model can bake a checkerboard into the image pixels. When true transparency is essential, request real alpha-channel transparency and explicitly prohibit checkerboards and transparency-grid patterns.

Good uses: hero/lifestyle imagery, abstract/pattern backgrounds, illustration-style benefit graphics, brand-consistent placeholder product shots.

## 3. Fallback

If you can't find or generate a fit, use `https://placehold.co/{width}x{height}` with sensible dimensions. Never leave `<Image>` with a broken/empty `src`.

## 4. next/image discipline (CLS/LCP/a11y)

- Use `next/image`, never a raw `<img>` (the Definition of Done greps for `<img`).
- Give every image **explicit dimensions** (`width`/`height`, or `fill` inside a sized, positioned container).
- Mark the LCP image (usually the hero) `priority`.
- Preserve a template's aspect-ratio container and `objectFit`/`objectPosition` when adapting.
- Meaningful `alt` on everything; decorative images get `alt=""`.

## 5. Image strategy per section

| Section             | Approach                                                                                            |
| ------------------- | --------------------------------------------------------------------------------------------------- |
| Hero                | User's best product photo or a generated lifestyle/hero image. Full-width, high-impact, `priority`. |
| Product showcase    | User product images (`find_assets`), multiple angles if available.                          |
| Benefits            | `lucide-react` icons, or generated illustration graphics.                                           |
| Testimonials        | Avatar placeholders (initials or generated); don't fabricate customer photos.                       |
| Editorial/deep-dive | User lifestyle photos, or generated contextual imagery.                                             |
| Collection/grid     | Product images from assets; fall back to generated product shots.                                   |
