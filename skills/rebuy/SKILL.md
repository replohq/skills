---
name: rebuy
title: Rebuy Recommendations
summary: Show Rebuy product recommendations on a page.
description: "REQUIRED before generating pages with Rebuy product recommendation data. Contains mandatory data loading patterns — code generated without reading this skill will use incorrect architecture."
---

# Rebuy Integration

## Data Loading Pattern

**Always use the prefetch pattern by default.** The only exception is when data must be loaded dynamically in response to a client-side input or interaction.

The pattern has two layers:

1. **Server component (page):** Uses `PrefetchedLoaders` to fetch data during SSR and seed the React Query cache via `HydrationBoundary`. This ensures the first paint includes the data with no loading state.
2. **Client component:** Uses a loader component (`RebuyRecommendationsLoader`) with `useSuspenseQuery` under the hood. On initial render it reads the prefetched cache — instant. On subsequent client-side navigations or prop changes, it fetches fresh data automatically.

### Example: Recommended Products Section with Prefetch

```tsx
// app/page.tsx (Server Component)
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";
import { RecommendedProducts } from "./RecommendedProducts";

export default function Page() {
  return (
    <PrefetchedLoaders
      queries={[
        {
          loaderKey: DATA_LOADER_KEYS.REBUY_RECOMMENDATIONS,
          args: { shopifyProductIds: "123456,789012", limit: 5 },
        },
      ]}
    >
      <RecommendedProducts />
    </PrefetchedLoaders>
  );
}
```

```tsx
// app/RecommendedProducts.tsx (Client Component)
"use client";

import Image from "next/image";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { RebuyRecommendationsLoader } from "@replohq/sdk/loaders/rebuy-recommendations-loader";

export function RecommendedProducts() {
  return (
    <RebuyRecommendationsLoader
      loaderKey={DATA_LOADER_KEYS.REBUY_RECOMMENDATIONS}
      shopifyProductIds="123456,789012"
      limit={5}
      fallback={<div>Loading recommendations...</div>}
    >
      {(data) => (
        <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
          {data.products.map((product) => (
            <div key={product.id} className="border rounded-lg p-4">
              {product.featuredImage && (
                <Image
                  src={product.featuredImage.url}
                  alt={product.title ?? ""}
                  width={product.featuredImage.width ?? 400}
                  height={product.featuredImage.height ?? 400}
                  className="w-full aspect-square object-cover"
                />
              )}
              <h3 className="mt-2 font-medium">{product.title}</h3>
              {product.variants?.[0]?.price && (
                <p className="text-sm text-gray-600">
                  ${product.variants[0].price.amount}
                </p>
              )}
            </div>
          ))}
        </div>
      )}
    </RebuyRecommendationsLoader>
  );
}
```

## Product Shape

The loader returns data in the canonical `Product` shape (partial). Guaranteed fields:

- `id` — Shopify product ID (numeric string)
- `images` — array of `{ url, altText, width, height }`

Optional fields (use optional chaining):

- `title` — product title
- `handle` — URL handle
- `vendor` — vendor name (may be null)
- `featuredImage` — `{ url, altText, width, height }` or null
- `variants` — array with `{ id, title, availableForSale, price: { amount, currencyCode }, compareAtPrice, sku }`
- `tags` — string array
- `availableForSale` — boolean

Always use optional chaining (`?.`) when accessing non-guaranteed fields.
