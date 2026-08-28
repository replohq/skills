# Video / autoplay on iOS

Read this when a background or hero `<video>` with autoplay won't play on
iPhone/iPad.

- **Symptom:** a background or hero `<video>` with autoplay does not play on iPhone/iPad.
- **Root cause:** iOS only autoplays video that is **muted** and set to play **inline** (otherwise it demands fullscreen or user interaction).
- **Fix:** `muted` + `playsInline` on the `<video>`; for maximum coverage also the legacy attributes `webkit-playsinline` (and `x5-playsinline` for some in-app WebViews).
- **Next.js/Tailwind:** `<video autoPlay muted playsInline loop … />`. React needs `playsInline` (camelCase); add the lowercase legacy attrs only if targeting old in-app browsers.
