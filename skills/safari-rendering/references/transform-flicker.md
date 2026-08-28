# Transform / compositing flicker in Safari

Read this when an animated element (marquee, carousel, anything with a looping
CSS transform) — especially one containing images — flickers in Safari when the
transform resets.

- **Symptom:** an animated element (marquee, carousel, anything with a looping CSS transform) — especially one containing images — flickers in Safari when the transform resets.
- **Root cause:** WebKit has compositing bugs translating elements that contain images.
- **Fix:** promote the element to its own GPU layer with `transform: translateZ(0)` / `translate3d(0,0,0)` and `backface-visibility: hidden`.
- **Next.js/Tailwind:** `transform-gpu backface-hidden` (or an inline `translateZ(0)`) on the flickering animated element.
