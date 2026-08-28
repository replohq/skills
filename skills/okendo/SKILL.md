---
name: okendo
title: Okendo Reviews
summary: Render Okendo reviews and star ratings on a page.
description: "REQUIRED before generating pages with Okendo review data. Contains mandatory data loading patterns — code generated without reading this skill will use incorrect architecture."
---

# Okendo Integration

## Data Loading Pattern

**Always use the prefetch pattern by default.** The only exception is when data must be loaded dynamically in response to a client-side input or interaction.

The pattern has two layers:

1. **Server component (page):** Uses `PrefetchedLoaders` to fetch data during SSR and seed the React Query cache via `HydrationBoundary`. This ensures the first paint includes the data with no loading state.
2. **Client component:** Uses a loader component (e.g., `OkendoReviewsLoader`) with `useSuspenseQuery` under the hood. On initial render it reads the prefetched cache — instant. On subsequent client-side navigations or prop changes, it fetches fresh data automatically.

### Discovering the Shopify Product ID

Okendo loaders require a **numeric Shopify product ID** (e.g. `"12345"`), not a Shopify GID. To get this:

1. Get the product's numeric ID from the Shopify admin (the number at the end of the product's admin URL), or ask a Replo session to look it up. Given a GID like `gid://shopify/Product/12345`, take the numeric portion.
2. Extract the numeric portion: `const numericId = gid.split("/").pop()`.
3. Pass the numeric ID to Okendo loader components as `shopifyProductId`.

### Example: Reviews Section with Prefetch

**Client component** (`app/components/ProductReviews.tsx`):

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { OkendoReviewsLoader } from "@replohq/sdk/loaders/okendo-reviews-loader";

export function ProductReviews({ shopifyProductId }: { shopifyProductId: string }) {
  return (
    <OkendoReviewsLoader
      loaderKey={DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEWS}
      shopifyProductId={shopifyProductId}
      fallback={<div>Loading reviews…</div>}
    >
      {({ reviews }) => (
        <div>
          <h2>{reviews.length} Reviews</h2>
          {reviews.map((review) => (
            <div key={review.id} className="border-b py-4">
              <div className="flex items-center gap-2">
                <span>{"★".repeat(review.rating ?? 0)}</span>
                <span className="font-semibold">{review.title}</span>
              </div>
              {review.reviewerName && (
                <p className="text-sm text-gray-500">by {review.reviewerName}</p>
              )}
              <p className="mt-2">{review.body}</p>
            </div>
          ))}
        </div>
      )}
    </OkendoReviewsLoader>
  );
}
```

**Server component** (`app/products/[handle]/page.tsx`):

```tsx
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductReviews } from "@/app/components/ProductReviews";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";

const SHOPIFY_PRODUCT_ID = "12345";

export default function ProductPage() {
  return (
    <PrefetchedLoaders
      queries={[
        {
          loaderKey: DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEWS,
          args: { shopifyProductId: SHOPIFY_PRODUCT_ID },
        },
      ]}
    >
      <ProductReviews shopifyProductId={SHOPIFY_PRODUCT_ID} />
    </PrefetchedLoaders>
  );
}
```

### Example: Star Rating Widget with Aggregate Data

**Client component** (`app/components/StarRating.tsx`):

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { OkendoReviewAggregateLoader } from "@replohq/sdk/loaders/okendo-review-aggregate-loader";

export function StarRating({ shopifyProductId }: { shopifyProductId: string }) {
  return (
    <OkendoReviewAggregateLoader
      loaderKey={DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEW_AGGREGATE}
      shopifyProductId={shopifyProductId}
      fallback={<div>Loading rating…</div>}
    >
      {(aggregate) => (
        <div className="flex items-center gap-2">
          <span className="text-yellow-500">
            {"★".repeat(Math.round(aggregate.reviewAverageValue))}
            {"☆".repeat(5 - Math.round(aggregate.reviewAverageValue))}
          </span>
          <span className="text-sm text-gray-600">
            {aggregate.reviewAverageValue.toFixed(1)} ({aggregate.reviewCount} reviews)
          </span>
        </div>
      )}
    </OkendoReviewAggregateLoader>
  );
}
```

**Server component** — prefetch alongside reviews or product data:

```tsx
<PrefetchedLoaders
  queries={[
    {
      loaderKey: DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEW_AGGREGATE,
      args: { shopifyProductId: SHOPIFY_PRODUCT_ID },
    },
    {
      loaderKey: DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEWS,
      args: { shopifyProductId: SHOPIFY_PRODUCT_ID },
    },
  ]}
>
  <StarRating shopifyProductId={SHOPIFY_PRODUCT_ID} />
  <ProductReviews shopifyProductId={SHOPIFY_PRODUCT_ID} />
</PrefetchedLoaders>
```

## Available Loaders

| Loader Key | Component | Import Path | Input |
|---|---|---|---|
| `DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEWS` | `OkendoReviewsLoader` | `@replohq/sdk/loaders/okendo-reviews-loader` | `{ shopifyProductId: string }` |
| `DATA_LOADER_KEYS.OKENDO_PRODUCT_REVIEW_AGGREGATE` | `OkendoReviewAggregateLoader` | `@replohq/sdk/loaders/okendo-review-aggregate-loader` | `{ shopifyProductId: string }` |

## Key Rules

- **Always prefer the prefetch pattern** (server component with `PrefetchedLoaders` wrapping a client loader component).
- Use `DATA_LOADER_KEYS` constants from `@replohq/sdk/loaders/loader-keys` — never raw strings.
- The `shopifyProductId` must be a **numeric** product ID, not a GID. Extract it from the GID if needed.
- Both Okendo loaders use the public Storefront API — no API key is needed at render time (only the `okendoUserId` from the project's connection, resolved server-side).
- Loader components use React Query hooks → they **must** live in `"use client"` files.
- `PrefetchedLoaders` is a **server component** → import it directly from `@replohq/sdk/loaders/prefetch-loaders`.
