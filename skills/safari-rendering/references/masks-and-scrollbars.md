# Loading shimmer / masks & scrollbars in Safari

Read this when a CSS mask animation (shimmer skeleton) doesn't animate in
Safari, or scrollbars show where you want them hidden.

- **Symptom:** a CSS mask animation (shimmer skeleton) doesn't animate, or scrollbars show where you want them hidden — in Safari.
- **Fix:** WebKit needs the `-webkit-`-prefixed variants: `-webkit-mask` / `-webkit-animation` for masked shimmer; `::-webkit-scrollbar { display: none }` (plus `scrollbar-width: none` for Firefox) to hide scrollbars.
