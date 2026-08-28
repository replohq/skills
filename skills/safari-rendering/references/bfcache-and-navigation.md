# Back-forward cache (bfcache) & navigation in Safari

Read this when a page comes back dead/non-interactive after the browser Back
button in Safari, or the scroll position is wrong after navigating back.

> On a Next.js page the App Router handles most navigation client-side, so this mainly matters for a
> hard back-navigation to a published page.

- **Symptom:** after pressing the browser Back button in Safari, the page comes back non-interactive (JS dead), or the scroll position is wrong.
- **Root cause:** Safari's aggressive bfcache serves a frozen static snapshot; it also doesn't auto-restore scroll the way other browsers do.
- **Fix:** on `pageshow`, if `event.persisted`, `window.location.reload()`. For scroll, save `scrollY` on `beforeunload` and restore it on load — for Safari only, since other browsers do it automatically.
- **Next.js:** apply the `pageshow`/`persisted` reload in a small client component if a user reports a dead page after Back on iOS.
