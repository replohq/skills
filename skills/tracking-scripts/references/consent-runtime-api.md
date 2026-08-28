# Consent runtime API

This reference covers the runtime surfaces for reading and updating consent: the
global `window.Replo.customerPrivacy` API (for plain on-page scripts), the
`visitorConsentCollected` DOM event, and the `useConsent()` React hook (for
components, including your own consent banner).

The consent categories are `necessary`, `analytics`, `marketing`, `preferences`,
and `sale_of_data` — these mirror Shopify's Customer Privacy API signals.
`necessary` is always granted and is never toggleable.

## `window.Replo.customerPrivacy`

Installed automatically by `<ReploScripts>` (present in the root layout whenever the runtime layer is set up),
so it is available at runtime on every page with no extra setup. Mirrors Shopify's
Customer Privacy API method names, under our own `window.Replo` namespace to avoid
colliding with Shopify's real API on Shopify-backed sites.

```ts
interface ReploCustomerPrivacyApi {
  // Per-category declaration: "yes" | "no" | "" ("" = visitor hasn't decided).
  currentVisitorConsent(): {
    analytics: "yes" | "no" | "";
    marketing: "yes" | "no" | "";
    preferences: "yes" | "no" | "";
    sale_of_data: "yes" | "no" | "";
  };

  // Booleans honoring the site's consent platform. With Replo's native banner,
  // all true when mode === "off" (no consent system at all). When Cookiebot or
  // another connected consent platform owns consent, these report that
  // platform's answer instead — denied until the visitor responds, even though
  // the native mode is off on such sites.
  analyticsProcessingAllowed(): boolean;
  marketingAllowed(): boolean;
  preferencesProcessingAllowed(): boolean;
  saleOfDataAllowed(): boolean;

  // True when Replo's banner should still be shown (mode on + visitor
  // undecided). Always false when Cookiebot or another connected consent
  // platform owns consent — it has its own banner.
  shouldShowBanner(): boolean;

  // Update a subset of categories; runs the optional callback once persisted.
  setTrackingConsent(
    consent: Partial<{
      analytics: boolean;
      marketing: boolean;
      preferences: boolean;
      sale_of_data: boolean;
    }>,
    callback?: () => void,
  ): void;
}
```

Example — a plain script that only initializes once analytics is allowed:

```js
function startTracking() {
  /* load / initialize the vendor */
}

if (window.Replo?.customerPrivacy?.analyticsProcessingAllowed()) {
  startTracking();
}
```

## `visitorConsentCollected` event

Dispatched on `document` every time consent changes. Use it to (re)initialize a
script when the visitor grants consent after the initial load.

```js
document.addEventListener("visitorConsentCollected", (event) => {
  const { analyticsAllowed, marketingAllowed, preferencesAllowed, saleOfDataAllowed } =
    event.detail;
  if (analyticsAllowed) {
    startTracking();
  }
});
```

`event.detail` shape:

```ts
{
  analyticsAllowed: boolean;
  marketingAllowed: boolean;
  preferencesAllowed: boolean;
  saleOfDataAllowed: boolean;
}
```

## `useConsent()` (React)

For client components. Import from `@replohq/sdk/consent/consent-store`. This is
what you drive your own consent banner with.

```ts
import { useConsent } from "@replohq/sdk/consent/consent-store";

const {
  mode,        // "off" | "simple" | "per-category"
  cmp,         // "replo" | "cookiebot" | "external" | undefined — which platform owns consent
  categories,  // { necessary, analytics, marketing, preferences, sale_of_data } booleans
  decidedAt,   // epoch ms when the visitor last decided, or null if undecided
  accept,      // () => void — grant all categories
  reject,      // () => void — deny all non-necessary categories
  updateCategories, // (next: Partial<categories>) => void — set a subset
} = useConsent({ version: 1 });
```

`version` is the site-owned consent policy version, stamped onto each
proof-of-consent record. Increment it whenever the banner copy or category set
changes. It defaults to 1 when omitted, but a banner should always declare it.

## Editing the consent banner

The banner is a normal component of the site at `components/consent/consent-banner.tsx`
(`<ConsentBanner>`), just like the cart UI (`components/cart/slideout-cart.tsx`).
Edit it directly to restyle it, or rebuild it from scratch following the shape
below. **Keep `<ReploScripts>` and the `data-replo-consent-mode` attribute in
the layout** — those are the consent engine; only the banner UI is yours to
change. (Consent state is seeded and recorded inside `@replohq/sdk`, so the layout
needs no seed `<script>`, project id, or analytics URL.)

**When you change the banner copy or the category set, increment the policy
version the banner passes to `useConsent({ version })`** — proof-of-consent
records carry it so each consent can be tied to the banner version the visitor
actually saw. The starter banner keeps it in a `CONSENT_POLICY_VERSION`
constant; any custom banner must pass one too.

```tsx
"use client";

import { useConsent } from "@replohq/sdk/consent/consent-store";

// Bump whenever the banner copy or category set changes.
const CONSENT_POLICY_VERSION = 1;

export function MyConsentBanner() {
  const consent = useConsent({ version: CONSENT_POLICY_VERSION });

  // Hide when consent is disabled or delegated to another platform (`cmp`
  // names the owning platform — "cookiebot" or "external" for a generic
  // consentPlatform), or once the visitor has decided.
  if (
    consent.cmp === "cookiebot" ||
    consent.cmp === "external" ||
    consent.mode === "off" ||
    consent.decidedAt !== null
  ) {
    return null;
  }

  return (
    <div role="dialog" aria-label="Cookie consent" className="fixed bottom-4 right-4 ...">
      <p>We use cookies to improve your experience.</p>
      <div className="flex gap-2">
        <button onClick={() => consent.accept()}>Accept all</button>
        <button onClick={() => consent.reject()}>Reject all</button>
        {consent.mode === "per-category" && (
          <button
            onClick={() =>
              consent.updateCategories({ analytics: true, marketing: false })
            }
          >
            Save choices
          </button>
        )}
      </div>
    </div>
  );
}
```

Notes:

- The banner must be a client component (`"use client"`) because it uses a hook.
- Don't gate the banner's own visibility on anything other than `mode` and
  `decidedAt` — `<ReploScripts>` handles gating the actual tracking scripts.
- After `accept` / `reject` / `updateCategories`, the consent store persists the
  choice to the `replo_consent` cookie and the `visitorConsentCollected` event
  fires automatically.
