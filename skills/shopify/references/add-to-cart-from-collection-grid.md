# Add to Cart from a Collection Grid

How to wire `useAddToCart` to product cards rendered from a `CollectionProductsLoader`.

## Rules

- The `merchandiseId` must be a variant GID (`product.variants[N].id`). **Never** pass `product.id` — that's the Product GID, not a variant GID, and Shopify will reject it.
- For a single-variant card, use `firstAvailableVariant.id` (or fall back to `product.variants?.[0]?.id` if nothing is in stock).
- For a multi-variant card with a variant picker, track the user-selected variant in local state and pass its `id`.
- No need to pass `merchandise` — the cart hook auto-resolves display data from the `CollectionProductsLoader` cache for optimistic UI.

## Example

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { useAddToCart } from "@replohq/sdk/cart/hooks/use-add-to-cart";
import { CollectionProductsLoader } from "@replohq/sdk/loaders/collection-products-loader";

export function CollectionGridWithCart({
  collectionGid,
}: {
  collectionGid: string;
}) {
  const { addToCart, isAdding } = useAddToCart();

  return (
    <CollectionProductsLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_COLLECTION_PRODUCTS}
      collectionGid={collectionGid}
      first={24}
      fallback={<div>Loading collection…</div>}
    >
      {({ products }) => (
        <ul>
          {products.map((product) => {
            const firstAvailableVariant =
              product.variants?.find((variant) => variant.availableForSale) ??
              product.variants?.[0];
            return (
              <li key={product.id}>
                <h3>{product.title}</h3>
                <button
                  disabled={
                    !firstAvailableVariant?.availableForSale || isAdding
                  }
                  onClick={() => {
                    if (firstAvailableVariant) {
                      addToCart([
                        {
                          merchandiseId: firstAvailableVariant.id,
                          quantity: 1,
                        },
                      ]);
                    }
                  }}
                >
                  {isAdding ? "Adding…" : "Add to Cart"}
                </button>
              </li>
            );
          })}
        </ul>
      )}
    </CollectionProductsLoader>
  );
}
```
