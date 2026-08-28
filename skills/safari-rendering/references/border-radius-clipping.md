# border-radius + overflow clipping in Safari

Read this when a child image/video/iframe embed doesn't get its corners clipped
by a parent's `border-radius` in Safari — the corners "stick out."

- **Symptom:** a child image/video/embed (e.g. a YouTube iframe) does not get its corners clipped by a parent's `border-radius` in Safari — the corners "stick out."
- **Root cause:** Safari fails to clip descendants to rounded corners in some configurations (transformed children, iframes).
- **Fix:** ensure `overflow: hidden` on the rounded container. If that isn't enough (transforms/embeds), add `-webkit-mask-image: -webkit-radial-gradient(white, black)` to force the clip. Note: don't add the mask when `overflow: hidden` is already present and a box-shadow needs to render (a mask clips the shadow too).
- **Next.js/Tailwind:** `rounded-* overflow-hidden` first; add the `-webkit-mask-image` inline style only if an embed/transform still bleeds past the corners.
