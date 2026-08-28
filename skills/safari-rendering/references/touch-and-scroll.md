# Touch / iOS interaction (gestures and modal scroll)

Read this for iOS touch behavior: carousel swipes stolen by page scroll, tap
highlights/callouts, or nested scrolling dead inside a modal.

## a. Carousel / swipe gestures

- **Symptom:** horizontal swipe on a carousel is stolen by the page's vertical scroll (or vice versa) on iOS; long-press shows the callout menu; tap highlights flash.
- **Fix:** `touch-action: pan-y` (for a horizontal carousel) to reserve the swipe axis; `-webkit-touch-callout: none`, `-webkit-tap-highlight-color: transparent`, `user-select: none`.

## b. Modal body-scroll-lock breaks nested scroll on iOS

- **Symptom:** a scrollable area inside an open modal can't be scrolled on iOS (works on Android).
- **Root cause:** `body-scroll-lock` calls `preventDefault` on `touchmove` by default, which kills nested scrolling in iOS Safari.
- **Fix:** allow `touchmove` for targets inside the modal body (`allowTouchMove: (target) => modalBody.contains(target)`). Also: closing an iOS modal that used `position: fixed` on the body can leave the page scrolled up — restore the scroll position on close.
