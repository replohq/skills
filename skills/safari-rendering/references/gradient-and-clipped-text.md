# Gradient / clipped text in Safari

Read this when gradient (background-clipped) text or a text-stroke renders as a
solid color in Safari.

- **Symptom:** text meant to show a gradient (background-clipped text) renders as solid color in Safari.
- **Root cause:** background-clip text requires `-webkit-` prefixes, and WebKit/Blink both have a bug where the `-webkit-background-clip` value assigned via a CSS custom property is not applied.
- **Fix:** `-webkit-background-clip: text; -webkit-text-fill-color: transparent` (plus `-moz-background-clip: text`). Set the literal value `text` directly, **not** through a CSS variable.
- **Next.js/Tailwind:** apply the `-webkit-` properties as an inline `style` (or a small utility class) with literal `text`, not via a `var()`.
