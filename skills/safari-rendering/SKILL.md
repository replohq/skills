---
name: safari-rendering
title: Fix Safari & Mobile Rendering
summary: Fix a page that renders correctly in preview but breaks on Safari or mobile.
description: 'Use when a page looks correct in the editor/preview or on the user''s laptop but wrong somewhere else — especially "only on mobile", "only on my phone/iPhone/iPad", "only on the live page", "only after publishing", "broken in Safari", "fine in Chrome but not Safari", images that collapse/squish/stretch, sections shorter than expected, content overflowing, rounded corners not clipping, a sticky/fixed bottom bar or footer/checkout/CTA that is cut off or won''t stay pinned to the bottom on mobile, content hidden behind the toolbar or home indicator, sticky headers jumping, gradient text showing solid, autoplay video not playing on mobile, or animation flicker. These are almost always WebKit (Safari/iOS) rendering differences, not editor bugs. Load before diagnosing or "fixing" a difference that only appears on a real device or the published URL.'
---

# Safari & Mobile Rendering Issues

## The core insight (read first)

**The editor preview and the live page do not render in the same browser.**

- It's very common for the **editor preview** and page editing to happen in **Chrome/Chromium (Blink)**, unless the user is using Safari or Firefox (uncommon).
- The **live/published page on a phone or tablet** renders in whatever browser engine the user uses to access it, and on **iOS/iPadOS every browser is WebKit** (Chrome for iOS, Firefox for iOS, in-app browsers like Instagram/TikTok all use WebKit under the hood). There is no non-WebKit browser on an iPhone.

So when a user says "it looks fine in the editor but broken on my phone", "only on mobile/iPhone/iPad", "only on the live page", or "works in Chrome, broken in Safari", the default hypothesis is a **WebKit-specific rendering difference** — not a Replo bug and not something you can reproduce by looking at the editor preview. If you don't see an obvious data/logic bug, you're most likely looking at Blink, they are looking at WebKit. Don't tell the user "it looks fine to me" from the editor.

## When to use

- A visual difference that appears **only on a real device**, **only on the published URL**, or **only in Safari** — not in the editor preview.
- Any symptom in the reference index below.

**Not for:** a difference you can already reproduce identically in Chrome (that's a normal layout/logic bug — fix it directly), or genuine responsive-design work (a layout simply too wide for a phone is a breakpoint problem, not a WebKit bug).

## Diagnose: confirm it is actually WebKit first

Before changing anything, establish that the difference is browser-engine-specific: **Get the exact conditions.** Which device, which browser, mobile vs. desktop, published vs. editor. "Broken on my phone" almost always means iOS Safari.

## Reference index

There are lots of categories of Webkit bugs, and has its own reference file with the symptom, the root cause, the concrete Next.js/Tailwind fix. **Read only the file(s) that match the symptom.** You usually don't need them all.

| Read this reference                                                             | When you see / are researching…                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [flex-grid-and-height.md](references/flex-grid-and-height.md)                   | Images or children squished, collapsed to a sliver, stretched, or the wrong height; a section shorter than expected; not filling available space; a grid child mis-sized inside a flex column; an `inset:0` overlay not covering.                                      |
| [image-sizing.md](references/image-sizing.md)                                   | An image renders too large/small, wrong aspect ratio or cropping, or collapsed — especially when only a height (not width) is set, or it's inside `<picture>`.                                                                                                         |
| [viewport-sticky-and-safe-area.md](references/viewport-sticky-and-safe-area.md) | A `100vh` section too tall or jumping as the toolbar shows/hides; a sticky/fixed bottom bar, footer, or CTA cut off, hidden behind the toolbar/home indicator, or not staying pinned; a full-height drawer drifting even at `100dvh`; `position: sticky` not sticking. |
| [border-radius-clipping.md](references/border-radius-clipping.md)               | Rounded corners not clipping a child image/video/iframe embed.                                                                                                                                                                                                         |
| [gradient-and-clipped-text.md](references/gradient-and-clipped-text.md)         | Gradient / background-clipped text or a text-stroke rendering as solid.                                                                                                                                                                                                |
| [form-controls.md](references/form-controls.md)                                 | A native `<select>` arrow or control chrome showing through; a full-width `<button>` not stretching.                                                                                                                                                                   |
| [ios-text-inflation.md](references/ios-text-inflation.md)                       | Text oversized on mobile after rotate/reflow.                                                                                                                                                                                                                          |
| [video-autoplay.md](references/video-autoplay.md)                               | A `<video>` autoplay / inline playback not working on iPhone/iPad.                                                                                                                                                                                                     |
| [transform-flicker.md](references/transform-flicker.md)                         | A marquee/carousel/animated element (esp. with images) flickering during CSS transforms.                                                                                                                                                                               |
| [touch-and-scroll.md](references/touch-and-scroll.md)                           | Carousel swipe stolen by page scroll, tap highlight/callout on iOS; nested scroll dead inside a modal/dialog on iOS.                                                                                                                                                   |
| [bfcache-and-navigation.md](references/bfcache-and-navigation.md)               | Page dead/non-interactive after the Back button; wrong scroll position after navigating back.                                                                                                                                                                          |
| [masks-and-scrollbars.md](references/masks-and-scrollbars.md)                   | A CSS mask / shimmer animation not showing; scrollbars not hidden.                                                                                                                                                                                                     |

## Applying fixes in a Replo Sites (Next.js) page

- **Prefer defensive, cross-browser CSS over user-agent sniffing.** Every fix in these references is a CSS/attribute change that is safe in all browsers. Published pages are server-rendered; branching layout on `navigator.userAgent` causes hydration mismatches. Apply the safe CSS everywhere, not "only in Safari."
- **Keep it token-first and on the Tailwind scale** — the fixes are extra utility classes (`min-h-0`, `shrink-0`, `aspect-[4/3]`, `overflow-hidden`), not raw colors or arbitrary values. Follow the **building-replo-pages** skill's conventions.
- **Some fixes are document-level, not per-element.** The iOS safe area needs a one-time `viewport-fit=cover` opt-in before `env(safe-area-inset-*)` does anything — see `viewport-sticky-and-safe-area.md`.
- **Don't strip a fix because "it looks redundant."** Several (especially the image min/max/height stack) look like duplicated properties but each is load-bearing in a specific Safari configuration; the reference notes which.

## Verification

A Safari fix is not done until you've seen it work **in WebKit**:

1. Apply the fix and confirm the page still renders and LSP/TypeScript is clean.
2. Have the user re-check on their real device — the same repro from the diagnose step. Confirm the symptom is gone. You may need to ask them to publish the project for this.
3. Confirm you did **not** regress Chrome/Blink (most fixes are neutral there, but percentage-height and `flex-basis` changes can shift Blink layout — eyeball both).
4. Only claim it's fixed after seeing confirmation.

## Common mistakes

- **Diagnosing from the editor preview.** The editor is Blink; the bug is WebKit
- **Using Chrome's device toolbar as "mobile."** It's still Blink and will not show WebKit bugs. It only helps with viewport-size (responsive) issues.
- **Treating a WebKit rendering bug as a data/content bug.** If the same data renders fine in Chrome, it's the engine, not the data.
- **UA-branching layout in a server-rendered page.** Causes hydration mismatches; use safe cross-browser CSS instead.

## Related Skills

- **building-replo-pages** — the general operating manual for building and editing Replo Sites pages; apply these Safari fixes within its token-first conventions.
- **publish** — the only way to deploy; a WebKit fix isn't real for the user until it's live, so verify on the published URL after publishing.
