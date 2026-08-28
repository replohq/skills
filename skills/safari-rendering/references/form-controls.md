# Form controls (`appearance`) in Safari

Read this when native Safari control chrome shows through — a `<select>` shows
its native arrow next to your custom one, or a full-width `<button>` won't
stretch.

- **Symptom:** native Safari control chrome shows through — a `<select>` shows its native arrow next to your custom one; a full-width `<button>` doesn't stretch.
- **Root causes:**
  - Safari's UA stylesheet sets `align-items: flex-start` on `<button>`, which overrides things like `width: 100%`.
  - Native `<select>`/control appearance needs to be reset before you can overlay a custom arrow.
- **Fixes:**
  - Button: `align-items: normal` on the button.
  - Select/control: `appearance: none` (`-webkit-appearance: none`) and render your own arrow overlay.
- **Next.js/Tailwind:** `appearance-none` on custom selects; `items-normal` (or an explicit `align-items: normal`) on a full-width button that won't stretch in Safari.
