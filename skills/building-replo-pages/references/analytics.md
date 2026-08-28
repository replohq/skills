# Analytics

Read this when building product/collection pages or adding event tracking. Analytics is wired up via `ReploProvider` in `app/layout.tsx` (`@replohq/sdk/analytics/`) — present in templates that ship the runtime layer; if the layout has no `ReploProvider`, add it first. Where the provider is present, events dispatch through consent-derived sinks, so firing via `useAnalytics()` is **safe** — you never check consent first.

## Auto-tracked — do NOT fire these manually

Firing these yourself causes duplicate tracking:

| Event            | Fired by            | When                                           |
| ---------------- | ------------------- | ---------------------------------------------- |
| `pageView`       | `AnalyticsProvider` | Every navigation (load, SPA nav, back/forward) |
| `addToCart`      | `CartProvider`      | Items added via `addToCart()`                  |
| `removeFromCart` | `CartProvider`      | Items removed via `removeCartItem()`           |
| `viewCart`       | `CartProvider`      | Cart opened via `openCart()`                   |
| `startCheckout`  | `SlideoutCart`      | Checkout button clicked                        |

## Fire these manually

**Product viewed** — dedicated hook on product detail pages:

```tsx
"use client";

import { useProductViewedAnalytics } from "@replohq/sdk/analytics/hooks/use-product-viewed-analytics";

function ProductPage({ product, selectedVariant }) {
  useProductViewedAnalytics({ product, selectedVariant });
  // product: { id: string; title: string; slug?: string; description?: string }
  // selectedVariant: { id: string; title?: string; sku?: string; price?: number }
}
```

**Collection viewed** — via `useAnalytics()`:

```tsx
"use client";

import { useAnalytics } from "@replohq/sdk/analytics/analytics-provider";

function CollectionPage({ collection }) {
  const analytics = useAnalytics();
  useEffect(() => {
    if (analytics) {
      void analytics.viewCollection({
        data: {
          collectionId: collection.id,
          collectionTitle: collection.title,
        },
      });
    }
  }, [analytics, collection.id, collection.title]);
}
```

**Reference:** `components/cart/slideout-cart.tsx` shows `useAnalytics()` firing `startCheckout`.

For marketing/analytics **pixels** (GA4, Meta, TikTok, …) and the cookie banner, use the **tracking-scripts** skill — don't hand-write `<script>` tags.
