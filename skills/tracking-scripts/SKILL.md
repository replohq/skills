---
name: tracking-scripts
title: Tracking Scripts & Consent
summary: Add analytics and marketing pixels so they respect cookie consent.
description: Use when the user asks to add, install, remove, or change an analytics or marketing tracking script, pixel, or tag — GA4 / Google Analytics, Google Tag Manager (GTM), Meta / Facebook pixel, TikTok, Pinterest, Reddit, Snapchat, Hotjar, Microsoft Clarity, Contentsquare, Triple Whale, Elevar, Segment, Northbeam, Converge, Cookiebot — or pastes a raw `<script>` tag and says "put this on my page", or mentions cookie consent, Cookiebot, a consent management platform (CMP), GDPR/CCPA compliance, customizing/restyling/replacing the cookie consent banner, or making a script respect consent at runtime (window.Replo.customerPrivacy / visitorConsentCollected).
---

# Tracking Scripts & Cookie Consent

Replo sites manage analytics/marketing scripts through a single consent-aware
registry so they only load with the visitor's consent (GDPR/CCPA). This skill is
the source of truth for adding or changing those scripts.

## Where managed scripts live

All managed tracking scripts are entries in the `scripts` array passed to the
single `<ReploScripts>` component in the site's root layout at
`app/layout.tsx`, in the site repo you cloned. Do not create or author files
under `src/app/`:

```tsx
import type { ReploScriptEntry } from "@replohq/sdk/consent/types";

const managedScripts: ReploScriptEntry[] = [
  { type: "GA4", identifier: "G-XXXXXXXXXX" },
  { type: "Meta", identifier: "123456789" },
  {
    type: "custom",
    id: "my-pixel",
    src: "https://example.com/p.js",
    requiredConsent: ["marketing"],
  },
];
```

The `ReploScriptEntry` type is imported from `@replohq/sdk/consent/types`. Do
**not** import it from `schemas/*` — a standalone site only resolves
`@replohq/sdk/*`.

`<ReploScripts>` (from `@replohq/sdk/consent`) handles gating and injection. You
do **not** write `<script>` tags for managed scripts — you only edit the array.

## Adding a known provider

Add one entry with the provider's PascalCase `type` and its ID/key as
`identifier`. Valid `type` values:

`GA4`, `GoogleTagManager`, `Meta`, `TikTok`, `Pinterest`, `Reddit`, `Snapchat`,
`Hotjar`, `MicrosoftClarity`, `Contentsquare`, `Segment`, `Northbeam`,
`Converge`, `Cookiebot` (a consent platform, not a tracker — see "Cookiebot"
below).

For Converge the `identifier` is the public token from the merchant's Client
Source (Converge → Data management → Event sources), which loads
`https://static.runconverge.com/pixels/{token}.js`. The snippet self-fires the
initial pageview; e-commerce funnel events and pageviews on SPA soft
navigations are fired by the Replo runtime's analytics sinks, so no extra
tracking code is needed.

For Contentsquare the `identifier` is the 13-character UXA tag ID (e.g.
`21d351bec970b`), which loads `https://t.contentsquare.net/uxa/{id}.js`.

```tsx
{ type: "Meta", identifier: "123456789" }
```

Some known providers (e.g. Triple Whale, Elevar) connect by pasting a dashboard
snippet instead of an ID — those use a generic `snippet` entry, see "Snippet
providers" below. `Cookiebot` is also a known provider but behaves differently —
it is a consent platform, not a tracker, see "Cookiebot" below.

Omit `requiredConsent` for identifier providers — each has a sensible default
consent category. Only set `requiredConsent` if the user explicitly wants to
recategorize it.

## Adding a custom script

For anything not in the known-provider list, add a `custom` entry. Custom entries
**must** set `requiredConsent` (there is no safe default), and **exactly one** of
`src` (external) or `body` (inline JS):

```tsx
{ type: "custom", id: "abacus", src: "https://abacus.io/t.js", requiredConsent: ["analytics"] }
{ type: "custom", id: "inline-1", body: "console.log('hi')", requiredConsent: ["marketing"] }
```

`requiredConsent` is an array of: `analytics`, `marketing`, `preferences`,
`sale_of_data`, or `necessary` (these mirror Shopify's Customer Privacy API
categories). If you cannot confidently pick a category, ask the user before
adding the entry.

## Snippet providers (Triple Whale, Elevar)

Some providers issue a unique snippet in their dashboard instead of an ID —
their script cannot be synthesized from an identifier. These use a generic
`snippet` entry: the pasted snippet rides in `body`, and `id` is the provider's
stable key (e.g. `"Triplewhale"`, `"Elevar"`). The runtime treats `id` as an
opaque dedupe key — there is no provider-specific `snippet` code, so a snippet
provider is pure config.

For every snippet provider: ask the user to copy the snippet from their
provider dashboard and paste it into the chat — never synthesize it yourself —
then add the snippet's JS (without the wrapping `<script>` tags) as a `snippet`
entry. `snippet` entries **must** set `requiredConsent` (the runtime has no
default). If the pasted snippet's wrapping tag is `<script type="module">`
(top-level `await` / `import()` in the body), the entry **must** also set
`module: true`, or the body is injected as a classic script and throws a
SyntaxError.

**Triple Whale** — no "Pixel ID"; the pixel is keyed by the store domain. The
user copies the versioned headless snippet from Triple Whale → Pixel Settings →
**Headless Installation**. Its snippet is a classic script (no `module` flag):

```tsx
{ type: "snippet", id: "Triplewhale", body: "<pasted snippet JS>", requiredConsent: ["analytics", "marketing"] }
```

Verify by running `TriplePixel('State')` in the browser console on the
published site — it should return `Ready`.

`Ready` is not enough on its own: Triple Whale only attributes sessions from
domains it knows about. After the snippet is in place, tell the user to register
every domain the site serves from (the custom subdomain, plus the
`replosites.com` staging domain if they test there) as a headless domain in
Triple Whale → **Pixel Settings**. Without it the pixel fires but Triple Whale
shows no sessions from the site.

**Elevar** — the snippet is unique per shop. The user copies it from the Elevar
dashboard → My Tracking → Sources → Shopify → headless installation step. It is
a `<script type="module">` snippet, so the entry must set `module: true`:

```tsx
{ type: "snippet", id: "Elevar", body: "<pasted snippet JS>", module: true, requiredConsent: ["analytics", "marketing"] }
```

Module snippets work under a consent platform too: the runtime injects a
blocked classic shim that re-creates the module tag when the platform
activates it, so `module: true` needs no special handling alongside Cookiebot.

Note for attribution: the base pixel only captures journeys. Purchases are
matched from order data on Triple Whale's side (automatic for Shopify-backed
stores via their Shopify integration).

## Cookiebot (consent management platform)

Cookiebot (by Usercentrics) is a **consent management platform**, not a tracker.
Add it like an identifier provider — its `identifier` is the **Domain Group ID
(CBID)**, a GUID from the Cookiebot dashboard (Settings → Your scripts):

```tsx
{ type: "Cookiebot", identifier: "00000000-0000-0000-0000-000000000000" }
```

**Check for an existing install first.** A user asking for Cookiebot often
already has it — a pasted `consent.cookiebot.com/uc.js` tag in the layout, or a
`custom` entry pointing at it. Search the layout for `consent.cookiebot.com`
before adding the provider entry, and if you find one, remove it in the same
edit: two loaders both set `window.Cookiebot` and fight over consent. Reuse the
CBID from their existing tag rather than asking for it again.

When a `Cookiebot` entry is present, Cookiebot — not Replo's native banner —
becomes the site's consent backend:

- Its loader (`https://consent.cookiebot.com/uc.js`, manual blocking mode) is
  injected unconditionally, because the CMP itself must run before consent.
- Every **other** managed script is injected but tagged with Cookiebot's
  blocking attributes (`type="text/plain"` + `data-cookieconsent="statistics"
|"marketing"|"preferences"`, mapped from each entry's `requiredConsent`), so
  Cookiebot only activates it once the visitor consents.
- Replo's native `<ReploScripts>` consent gate is bypassed for those scripts.
- Replo's **own** tracking follows Cookiebot too: the first-party pixel and
  analytics sinks read Cookiebot's answer instead of the native banner's, so
  they stay blocked until the visitor consents to statistics.

Because Cookiebot owns consent, the site's native consent **mode must be `off`**
(otherwise visitors see two banners). Setting it is part of installing any
consent platform: confirm with the user, then set
`data-replo-consent-mode="off"` — this is the one exception to the "don't
change the consent mode" rule below. Any
**raw/unmanaged** `<script>` in the layout bypasses Cookiebot's per-script tag
and runs before consent — convert it to a managed entry so Cookiebot can gate
it. The Scripts & Pixels miniapp turns the native mode off automatically when
Cookiebot is connected there, and flags both of these problems if they appear.

## Other consent platforms (generic `consentPlatform` entries)

Consent platforms other than Cookiebot (Termly, CookieYes, OneTrust, ...) that
work the same way — a loader script plus per-tag manual blocking — connect with
a generic entry. Derive two things from the platform's "manual blocking" docs:
its loader `<script>` tag, and the attribute it reads on blocked tags plus its
category token vocabulary:

```tsx
{
  type: "consentPlatform",
  id: "termly",
  src: "https://app.termly.io/resource-blocker/UUID",
  attributes: { "data-website-uuid": "UUID" },   // the loader tag's attributes
  blockingAttribute: {
    name: "data-categories",                      // attribute the platform reads
    values: {                                     // Replo category -> platform token
      analytics: "analytics",
      marketing: "advertising",
      preferences: "preferences",
      sale_of_data: "advertising",
    },
  },
}
```

Its presence delegates consent exactly like Cookiebot: every other managed
script is injected blocked (`type="text/plain"` + the annotation above), the
loader injects after them so the platform's startup scan sees the full set, and
Replo's own analytics rides the same blocking. Blocking never depends on the
`values` map being right — an unmapped category still blocks (fails closed), so
a wrong mapping shows up as a script that never activates, not as tracking
before consent. Set the native consent mode to `off` as part of the install,
same as Cookiebot. Do NOT guess tokens: if the platform's docs don't describe
per-tag manual blocking, tell the user the platform isn't supported rather than
inventing attributes.

## Removing a script

Delete its entry from the `scripts` array. Don't touch other entries.

## Raw scripts the user insists on pasting directly

If the user explicitly says to paste a raw `<script>` into `layout.tsx` (not as a
managed entry), you may do it — but you **must** warn them first:

> This script will load for every visitor regardless of cookie consent, so it
> won't be covered by GDPR/CCPA consent. Want me to add it as a managed script
> instead so it respects consent?

Only add the raw tag if they confirm after the warning. Prefer the managed array.
If the script can't be a managed entry but should still respect consent, offer to
gate it at runtime with the `window.Replo.customerPrivacy` API (see "Runtime
consent API" below) instead of letting it load unconditionally.

## Customizing the consent banner

The cookie banner is a **normal, editable component of the site** at
`components/consent/consent-banner.tsx` (in a subdirectory, so it is not a
surfaced Site Component) (`<ConsentBanner>`), exactly like the
cart UI (`components/cart/slideout-cart.tsx`). To restyle it, change its copy,
or rebuild it entirely, just edit that file — use the site's design tokens and
`components/ui/` primitives like any other component.

It's driven by the `useConsent({ version })` hook from
`@replohq/sdk/consent/consent-store`, which exposes `mode`, `categories`,
`decidedAt`, and `accept()` / `reject()` / `updateCategories()`. Keep
`<ReploScripts>` and the `data-replo-consent-mode` attribute in place — those
are the consent engine; only the banner UI is yours to change. (The consent
state is seeded and recorded entirely inside `@replohq/sdk` — no seed `<script>`
or env wiring belongs in `layout.tsx`.) See
[consent-runtime-api.md](references/consent-runtime-api.md) for the hook shape
and a worked example.

**Whenever you change the banner's copy or the set of categories it offers,
increment the policy version the banner passes to `useConsent({ version })`**
(the starter banner keeps it in a `CONSENT_POLICY_VERSION` constant) — it is
stamped onto each visitor's proof-of-consent record so past consents can be
tied to the exact banner version they saw.

## Runtime consent API (for scripts that must self-gate)

Scripts can read consent and react to changes at runtime via the global
`window.Replo.customerPrivacy` API and the `visitorConsentCollected` DOM event
(this mirrors Shopify's Customer Privacy API). This is the correct way to make a
**raw / custom** script consent-aware when it must stay on the page but only act
once consent is granted:

```js
if (window.Replo?.customerPrivacy?.analyticsProcessingAllowed()) {
  // safe to track
}
document.addEventListener("visitorConsentCollected", (event) => {
  if (event.detail.analyticsAllowed) {
    /* (re)initialize tracking */
  }
});
```

Managed entries in the `scripts` array do **not** need this — `<ReploScripts>`
already gates them at injection. Full surface is documented in
[consent-runtime-api.md](references/consent-runtime-api.md).

## What not to do

- Don't add `<script>`/`<style>`/`<link>` tracking tags to `layout.tsx` JSX for a
  known provider or a consent-categorizable script — use the `scripts` array.
- Don't remove `<ReploScripts>` or the `data-replo-consent-mode` attribute on
  `<html>` — together these are the consent enforcement layer. (The
  `<ConsentBanner>` component _is_ editable — see
  "Customizing the consent banner" below.)
- Don't change the consent **mode** (the `data-replo-consent-mode` value, or the
  `CONSENT_MODE` constant on older sites) yourself — only the customer changes it
  via the Scripts & Pixels miniapp. One exception: when connecting Cookiebot, set
  the mode to `off` after confirming with the user (see "Cookiebot").
- Don't forget to bump the banner's `useConsent({ version })` policy version
  when you change the banner copy or category set (see
  "Customizing the consent banner").
- Don't invent provider `type` values; use the exact PascalCase keys above.
- Don't set `requiredConsent` on known providers unless explicitly asked.
- Don't put both `src` and `body` on a custom entry.

## Quick reference

| User intent | Action |
| --- | --- |
| Add GA4 / Meta / TikTok / etc. | Add `{ type, identifier }` to the `scripts` array |
| Add Converge | Add `{ type: "Converge", identifier: "<public token>" }` (see "Adding a known provider") |
| Add an arbitrary tracking script | Add `{ type: "custom", id, src \| body, requiredConsent }` |
| Add Triple Whale | Ask for the dashboard headless snippet; add `{ type: "snippet", id: "Triplewhale", body, requiredConsent }`; tell the user to register the site's domains as headless domains in Triple Whale (see "Snippet providers") |
| Add Elevar | Ask for the dashboard headless snippet; add `{ type: "snippet", id: "Elevar", body, module: true, requiredConsent }` (see "Snippet providers") |
| Add Cookiebot | Check for an existing loader first; add `{ type: "Cookiebot", identifier: "<CBID GUID>" }` and set consent mode to `off` (see "Cookiebot") |
| Add another consent platform | Add a `consentPlatform` entry built from its manual-blocking docs (see "Other consent platforms") |
| Remove a tracker | Delete its entry from the `scripts` array |
| "Just paste this script in" | Warn it won't be consent-managed; offer a managed entry; only add raw if confirmed |
| Turn the consent banner on/off | Direct them to the Scripts & Pixels miniapp (consent mode) |
| Restyle / rebuild the cookie banner | Edit `components/consent/consent-banner.tsx` directly (driven by `useConsent()`) |
| Make a raw/custom script respect consent at runtime | Gate it with `window.Replo.customerPrivacy` + the `visitorConsentCollected` event |
