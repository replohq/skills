# Viewport height, sticky-to-bottom bars, and the iOS safe area

Read this for anything meant to fill the screen or pin to the bottom on mobile
Safari: cart drawers, sticky checkout/CTA bars, mobile nav bars, full-screen
overlays. `position: sticky` not stickingis at the end (section `d`).

## a. Viewport height (`100vh`) on iOS

- **Symptom:** a `height: 100vh` section is taller than the visible screen and/or jumps as the Safari toolbar shows and hides while scrolling.
- **Root cause:** iOS Safari's `100vh` is the _largest_ viewport (toolbar hidden), so content is cut off / shifts when the bar is visible.
- **Fix:** use the small/dynamic viewport units `100svh` or `100dvh` (or the WebKit `-webkit-fill-available` pattern) instead of `100vh`.
- **Next.js/Tailwind:** `min-h-[100svh]` / `h-[100dvh]` rather than `min-h-screen`/`h-screen` for full-height hero sections that must fit an iPhone.

## b. Full-height overlays/drawers: anchor both edges instead of setting a viewport height

- **Symptom:** a full-height drawer or overlay (e.g. a cart drawer) is slightly too tall/short on mobile Safari, and its bottom content drifts under the toolbar as the toolbar expands/collapses — even when sized with `100dvh`.
- **Root cause:** Safari's bottom toolbar expands and collapses as you scroll or tap, and an element sized with a _computed_ height like `100vh`/`100dvh` does not always keep pace with that resize in real time on iOS. A dynamic viewport unit is still a number the layout has to recompute; it lags the actual visible viewport.
- **Fix:** don't compute the height from a viewport unit at all. Anchor the element to **both** `top: 0` and `bottom: 0` (position `fixed`/`absolute`) and let the browser resolve the real height itself, so it tracks the live visible viewport instead of a snapshot of it.
- **Next.js/Tailwind:** `fixed inset-y-0` (or `top-0 bottom-0`) on the drawer/overlay instead of `h-[100dvh]`. Keep `dvh`/`svh` for ordinary in-flow full-height sections (a); reserve the both-edges anchor for pinned overlays.

## c. Sticky/fixed bottom bars cut off by the home indicator (the iOS safe area)

- **Symptom:** a bar pinned to the bottom on mobile Safari (checkout/promo footer, sticky "Add to cart" CTA, mobile bottom nav) is partly cut off, sits under the home indicator, or doesn't always stay flush to the bottom.
- **Root cause:** the iPhone home indicator (and rounded corners / notch) carve out a **safe area** at the very bottom of the screen. Content flush to the bottom edge reads as cut off unless you explicitly pad for it. The CSS environment variable `env(safe-area-inset-bottom)` gives that inset — but it reads as `0` unless the page opts in with the viewport meta `viewport-fit=cover`. Without opting in, Safari reports no inset and you can't pad correctly.
- **Fix (two required halves):**
  1. **Opt into the safe area at the document level:** set `viewport-fit=cover` in the viewport meta so Safari reports real safe-area values instead of `0`.
  2. **Pad the pinned element** with `env(safe-area-inset-bottom)` (usually added to its bottom padding) so it always clears the home indicator and any toolbar chrome, no matter how the toolbar expands or collapses.
- **Next.js:** set it once in the App Router viewport export — `export const viewport: Viewport = { viewportFit: "cover" }` (this emits `viewport-fit=cover` into the `<meta name="viewport">`). Then on the bar: `paddingBottom: "env(safe-area-inset-bottom)"`, or Tailwind arbitrary `pb-[env(safe-area-inset-bottom)]` (add to existing padding, e.g. `pb-[calc(1rem+env(safe-area-inset-bottom))]`). The `viewport-fit=cover` opt-in is easy to forget and is the reason `env(safe-area-inset-*)` "does nothing" — check it first when safe-area padding appears to have no effect.
- **Together (b + c):** a bottom-pinned bar that "sometimes isn't stuck / is cut off" on iOS almost always needs all three: `viewport-fit=cover`, the both-edges anchor (or a `sticky`/`fixed` bottom that isn't fighting a viewport-unit height), and `env(safe-area-inset-bottom)` padding. This exact combination fixed a cart checkout/promo footer that was cut off behind the toolbar/home indicator on mobile Safari.

## d. `position: sticky` silently not sticking

- **Symptom:** an element with `position: sticky` (top or bottom) never sticks — it just scrolls away.
- **Root cause:** `sticky` is disabled the moment **any** ancestor between it and its scroll container has `overflow` set to `hidden`, `auto`, `scroll`, or `clip` (that ancestor becomes the scroll container and clips the stick). It also needs the sticky direction's offset (`top`/`bottom`) set. This is technically cross-browser, but it bites most on mobile Safari and is a frequent cause of "sticky doesn't work only on my phone" once combined with a–c.
- **Fix:** ensure no ancestor up the chain clips overflow (a stray `overflow-hidden` added for a border-radius fix is a common culprit — see `border-radius-clipping.md`), and set the offset (`top-0` or `bottom-0`). For a bottom sticky bar, combine with the safe-area padding above.
- **Next.js/Tailwind:** `sticky bottom-0 pb-[env(safe-area-inset-bottom)]`, and audit ancestors for `overflow-hidden`/`overflow-auto`.
