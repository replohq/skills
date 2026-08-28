# Site routing

Read this when adding, editing, removing, or debugging a site's redirects — exact-page vs whole-subtree, capturing sub-paths, priority, status codes, and custom-middleware safety.

Replo Sites express redirects as `reploRedirect(...)` calls in the selected site's root `middleware.ts`. The Website Builder reads every redirect in that file, editing the ones whose source, destination, and status are literal values and showing the rest read-only.

This skill covers `middleware.ts` only. It does not audit the rest of the application for redirect behavior.

## Two kinds of redirect

Every redirect in `middleware.ts` is one of two things:

- **A Replo-supported redirect:** a `reploRedirect(...)` call whose source, destination, and status are literal values. The Website Builder can edit these.
- **A custom redirect:** anything else in the file that sends a response elsewhere — `NextResponse.redirect`, `Response.redirect`, `redirect()`, `permanentRedirect()`, or an explicit 3xx response with a `Location` header. These are typically conditional on authentication, cookies, headers, request method, locale, or feature flags, or compute their destination at runtime. Leave them exactly as they are; the Website Builder shows them read-only.

A `reploLocaleRouting(...)` call and the `localeRouting` argument to `evaluateRouting` are locale routing, not a redirect — leave them alone, do not classify them as custom redirects, and see `references/site-i18n.md` for their rules.

Preserve custom redirects and unrelated middleware behavior. If a redirect is ambiguous or cannot be changed while preserving matching, status, query, and execution order, change nothing and report it. Do not create `proxy.ts` for Replo redirects; the runtime uses `middleware.ts` for this feature.

## Rule format

Every rule is a `reploRedirect(...)` call with a `source`, a `destination`, and a `status`. The `source` is a path whose `:name` segments capture instead of matching literally, and those captures substitute into the destination:

- `/old-page` — exactly that page.
- `/old-blog/:rest*` — `/old-blog` and everything under it, capturing the rest.
- `/blog/:year/:slug` — two required segments, each captured by name.
- `/docs/:page?` — `/docs`, with or without one more segment.

`:name*` takes the remainder of the path, `:name+` one or more segments, `:name?` an optional single segment, and a bare `:name` exactly one. Write `/old-blog/:rest*`, never `/old-blog/*` — a bare `*` captures nothing and is rejected.

To carry the matched sub-path onto the destination, reuse the capture there: `source: "/old-blog/:rest*"` with `destination: "/blog/:rest*"` sends `/old-blog/post` to `/blog/post`. Omit it to send every match to one place.

A file with one exact-page rule and one capture rule:

```ts
import type { NextRequest } from "next/server";

import { evaluateRouting } from "@replohq/sdk/routing/evaluate";
import { reploRedirect } from "@replohq/sdk/routing/rules";

const rules = [
  reploRedirect({ source: "/old-page", destination: "/new-page", status: 308 }),
  reploRedirect({
    source: "/old-blog/:rest*",
    destination: "/blog/:rest*",
    status: 308,
  }),
];

export function middleware(request: NextRequest) {
  return evaluateRouting({ request, rules });
}

// The matcher must remain static so Next.js can detect it at build time.
export const config = {
  matcher: ["/((?!_next/).*)"],
};
```

On a site with locale routing configured, every rule is tried against the raw request path first and then against the locale-stripped path, so write sources against logical paths (`/old-page`) and they apply in every locale, with same-site destinations locale-prefixed automatically. A locale-prefixed source (`/en-CA/something`) matches exactly that locale and no other. See `references/site-i18n.md`.

Sources must start with `/`. Destinations may be another site path or a complete HTTP(S) URL, and may carry a query string. A same-origin destination preserves the incoming query string when it does not define one; a cross-origin destination drops it unless it defines its own.

## Priority

Array order is evaluation order and the first matching rule wins.

- A `:rest*` source matches its base path with or without a trailing slash and every descendant: `{ source: "/blogs/:rest*" }` matches `/blogs`, `/blogs/`, and `/blogs/2026/post`, but not `/blog` or `/blogs-old`.
- Put exact or narrower exceptions before their catch-all fallback. In the reverse order the catch-all rule shadows them, and the Website Builder blocks saving on that conflict.
- A destination that its own source matches loops forever and is rejected. If an absolute destination resolves to the current request at runtime, that first match is consumed and the request renders normally; later rules do not run.

## Status codes

- `308` is the default permanent redirect and preserves the request method.
- `307` is temporary and preserves the request method.
- `301` and `302` are legacy permanent and temporary redirects. Use them only when the user explicitly needs those semantics.

## Safe edits

- Do not add stable IDs, enabled flags, or a separate JSON registry. The rule's source, destination, status, and array position are the complete persisted rule.
- Do not generate dynamic matcher values. Next.js must statically analyze the literal `config.matcher` value.

## Rules for every redirect edit

- Edit only `middleware.ts`. Do not migrate redirects from other site files.
- Preserve conditional redirects and unrelated middleware behavior; never drop a redirect you were not asked to change.
- If the file's redirect behavior is ambiguous or cannot be preserved, change nothing and report the conflict.

## Verification

After editing, reread every changed file, then verify against the running site:

1. Each source returns the configured redirect status and `Location` header.
2. Each destination route still renders normally.
3. Representative non-matching and conditional routes are unchanged.
4. Query strings follow the preservation rule above.
5. If rules overlap, the first rule wins; reorder and recheck when priority changes.
6. For a `:rest*` source, check the base path, its trailing-slash form, a nested path, and a similarly prefixed non-match. When the destination carries the capture, confirm the sub-path is preserved; when it does not, confirm every match lands on the destination.

For removal, confirm the deleted rule no longer redirects and that normal routes still render.
