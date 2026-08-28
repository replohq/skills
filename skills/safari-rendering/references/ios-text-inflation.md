# iOS text auto-inflation

Read this when text looks oversized on mobile — especially after rotating the
device or when a narrow column reflows — only on iOS Safari.

- **Symptom:** text looks oversized on mobile, especially after rotating the device or when a narrow column reflows — only on iOS Safari.
- **Root cause:** iOS Safari "text size adjust" auto-inflates font sizes it thinks are too small for the viewport.
- **Fix:** `-webkit-text-size-adjust: none; text-size-adjust: none` (and `-moz-` for completeness).
- See https://kilianvalkhof.com/2022/css-html/your-css-reset-needs-text-size-adjust-probably/ for more info
- **Next.js/Tailwind:** add `text-size-adjust: none` in the site's `globals.css` reset (a good default for all Replo Sites).
