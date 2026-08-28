---
name: shopify
title: Shopify Data & Cart
summary: Load a store's catalog, metafields, and cart from Shopify.
description: "REQUIRED before using any Shopify tools or generating pages with product data. Contains mandatory data loading patterns — code generated without reading this skill will use incorrect architecture. Use when user asks about Shopify, Shop Pay, Shopify products, metafields or metaobjects, etc"
---

# Shopify Storefront Data & Cart

## Reading the store's catalog and custom data

The loaders below render Shopify data on your pages and need no Shopify
credentials — the site resolves them server-side. **Discovery** reads, though —
searching products, listing metafield or metaobject definitions, inspecting a
real metafield value — run against the store's Admin API, which is not on the
public surface. Two ways to get those answers:

- **Ask a Replo session.** Prompt `start_agent_session` with what you need, e.g.
  "List the metafield definitions for products on the connected Shopify store,
  and show me a real value for each."
- **Ask the user.** Product GIDs, handles, and metafield namespace/key pairs are
  all visible in the Shopify admin.

For Replo-managed products (a separate catalog from Shopify), use the
`find_products` and `get_product` tools directly.

## Translating Shopify catalog content

When the site uses locale routing / dictionaries **and** Shopify product or collection loaders, read [translating-shopify-content.md](references/translating-shopify-content.md). Dictionary files do not translate Shopify titles, descriptions, or metafields — pass `language` on the loaders.

## File Structure

Data loaders live in the `@replohq/sdk` package under `loaders/`:

- **Generic infrastructure:** `@replohq/sdk/loaders/invoke-loader.ts`, `@replohq/sdk/loaders/prefetch-loaders.tsx`
- **Loader components:** `@replohq/sdk/loaders/product-loader.tsx`, `@replohq/sdk/loaders/collection-loader.tsx`, `@replohq/sdk/loaders/collection-products-loader.tsx`

Import each loader component directly from its own file (e.g. `@replohq/sdk/loaders/product-loader`).
Import `PrefetchedLoaders` (server component) directly from `@replohq/sdk/loaders/prefetch-loaders`.

Loader components are generic — they work with any integration. The `loaderKey` prop tells the dependency resolver which integration handler to use. Always import constants from `@replohq/sdk/loaders/loader-keys`:

- **`DATA_LOADER_KEYS`** for loader keys (e.g., `DATA_LOADER_KEYS.SHOPIFY_PRODUCT`) — used in `loaderKey` props and `PrefetchedLoaders` queries.

React Query cache keys are derived internally as `[loaderKey, args]` — you never need to specify them manually.

Never use raw strings for loader keys.

## Non-Active Products

Search tools return every product, but loaders only render `ACTIVE` products published to the Online Store — a `DRAFT` or `ARCHIVED` product errors in pages (`UNLISTED` still renders when referenced directly by GID or handle). You can still build pages with non-active products; just tell the user the page won't show real data until they publish the product.

## Data Loading Pattern

**Always use the prefetch pattern by default.** The only exception is when data must be loaded dynamically in response to a client-side input or interaction.

The pattern has two layers:

1. **Server component (page):** Uses `PrefetchedLoaders` to fetch data during SSR and seed the React Query cache via `HydrationBoundary`. This ensures the first paint includes the data with no loading state.
2. **Client component:** Uses a loader component (e.g., `ProductLoader`) with `useSuspenseQuery` under the hood. On initial render it reads the prefetched cache — instant. On subsequent client-side navigations or prop changes, it fetches fresh data automatically.

### Example: Product Page with Prefetch (GID-based)

This is the default pattern. Use the product's Shopify GID (returned by search tools as `id`) to load the product via `productId`. The file locations below mirror the existing scaffold structure: `page.tsx` lives in the route folder, and client components live in `app/components/`. Build the client component first.

**Client component** (`app/components/FeaturedProductDetail.tsx`):

```tsx
"use client";

import Image from "next/image";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

export function FeaturedProductDetail({ productId }: { productId: string }) {
  return (
    <ProductLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}
      productId={productId}
      fallback={<div>Loading product…</div>}
    >
      {(product) => (
        <div>
          <h1>{product.title}</h1>
          {product.featuredImage && (
            <Image
              src={product.featuredImage.url}
              alt={product.featuredImage.altText ?? ""}
              width={product.featuredImage.width ?? 800}
              height={product.featuredImage.height ?? 800}
            />
          )}
          <p>{product.availableForSale ? "In stock" : "Out of stock"}</p>
          <ul>
            {product.variants.map((variant) => (
              <li key={variant.id}>
                {variant.title} — {variant.price.amount}{" "}
                {variant.price.currencyCode}
              </li>
            ))}
          </ul>
        </div>
      )}
    </ProductLoader>
  );
}
```

**Server component** (`app/products/featured/page.tsx`):

```tsx
import { FeaturedProductDetail } from "@/app/components/FeaturedProductDetail";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";

const FEATURED_PRODUCT_GID = "gid://shopify/Product/123456789";

export default function FeaturedProductPage() {
  return (
    <PrefetchedLoaders
      queries={[
        {
          loaderKey: DATA_LOADER_KEYS.SHOPIFY_PRODUCT,
          args: { productId: FEATURED_PRODUCT_GID },
        },
      ]}
    >
      <FeaturedProductDetail productId={FEATURED_PRODUCT_GID} />
    </PrefetchedLoaders>
  );
}
```

### Alternative: Dynamic Route with Slug

Use this pattern **only** when the product is determined dynamically from the URL — for example, a dynamic page that can render any product by its URL slug. The folder name uses Next.js dynamic route syntax where brackets are literal in the folder name (the folder is literally named `[slug]`), and Next.js maps the URL segment to `params.slug` at runtime. For example, visiting `/products/cool-sneakers` passes `"cool-sneakers"` as `params.slug`. Pass that value to the loader as `handle` (Shopify resolves products by handle); the `[slug]` segment name is the standard dynamic-route shape shared with Replo Products.

**Server component** (`app/products/[slug]/page.tsx`):

```tsx
import { ProductDetail } from "@/app/components/ProductDetail";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";

export default function ProductPage({ params }: { params: { slug: string } }) {
  return (
    <PrefetchedLoaders
      queries={[
        {
          loaderKey: DATA_LOADER_KEYS.SHOPIFY_PRODUCT,
          args: { handle: params.slug },
        },
      ]}
    >
      <ProductDetail handle={params.slug} />
    </PrefetchedLoaders>
  );
}
```

**Client component** (`app/components/ProductDetail.tsx`):

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

export function ProductDetail({ handle }: { handle: string }) {
  return (
    <ProductLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}
      handle={handle}
      fallback={<div>Loading product…</div>}
    >
      {(product) => (
        <div>
          <h1>{product.title}</h1>
          {/* ... render product details ... */}
        </div>
      )}
    </ProductLoader>
  );
}
```

**Previewing and verifying a dynamic route:** the literal bracket URL (`/products/[slug]`) is not a real page, and a dev server may substitute sample data for it — seeing "Sample product" there is expected and does not prove real data works. Always verify with a concrete URL built from a real handle (from the connected store or a listing-page link), never the bracket route.

### Key rules for the pattern

- `PrefetchedLoaders` derives query keys internally as `[loaderKey, args]`, matching what loader components use. You only need to provide `loaderKey` and `args`.
- Every loader component renders via a **render-prop** (`children` is a function receiving typed data).
- Loader components use React Query hooks → they **must** live in `"use client"` files. Never pass a render-prop function from a server component.
- `PrefetchedLoaders` is a **server component** → import it directly from `@replohq/sdk/loaders/prefetch-loaders`, never through a barrel file that also exports client code.
- Provide a `fallback` prop for the loading state that displays while data is being fetched on client-side transitions.

### Collection Grids (CollectionProductsLoader)

`CollectionProductsLoader` returns each product with enough data to render variant-aware cards without a per-product fan-out. For every product you get:

- `minPrice` and `maxPrice` — render `$X` or `$X – $Y` price ranges.
- `compareAtMinPrice` and `compareAtMaxPrice` — render strike-through pricing. Both fields are always present; Storefront returns `amount: "0.0"` when there is no compare-at price, so check `Number(compareAtMinPrice.amount) > 0` before rendering.
- `options` and `options[].optionValues` (with swatch color / image) — render variant pickers and swatches on the card.
- `variants` — per-variant `availableForSale`, `sku`, `price`, `compareAtPrice`, `image`, `selectedOptions`. Drive sold-out states and add-to-cart buttons from these.

The product-level `availableForSale` is `true` whenever **any** variant is purchasable, so always inspect `variants[i].availableForSale` for accurate sold-out UI.

Variant lists are capped at the first 100 variants per product; products with more variants will trigger a server-side warning and the extras will be omitted. Use `ProductLoader` for products that need every variant.

```tsx
"use client";

import { CollectionProductsLoader } from "@replohq/sdk/loaders/collection-products-loader";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";

export function CollectionGrid({ collectionGid }: { collectionGid: string }) {
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
            const hasCompareAt =
              product.compareAtMinPrice != null &&
              Number(product.compareAtMinPrice.amount) > 0;
            const priceLabel =
              product.minPrice &&
              product.maxPrice &&
              product.minPrice.amount !== product.maxPrice.amount
                ? `${product.minPrice.amount} – ${product.maxPrice.amount}`
                : product.minPrice?.amount;
            return (
              <li key={product.id}>
                <h3>{product.title}</h3>
                <p>
                  {hasCompareAt && <s>{product.compareAtMinPrice!.amount}</s>}{" "}
                  {priceLabel}
                </p>
                {firstAvailableVariant == null && <span>Sold out</span>}
              </li>
            );
          })}
        </ul>
      )}
    </CollectionProductsLoader>
  );
}
```

## Unified Object Types

Product, Collection, and Variant are **canonical types** shared across the entire system. The same `Product` type is returned by search tools (sparsely populated) and data loaders (fully populated). This means you never need to convert between different representations.

A Replo session can report the available object types for the connected store. Each entry provides:

- `objectTypeKey` — canonical type name (e.g., `"product"`)
- `tags` — categories used for browsing and grouping objects
- `searchToolKey` — which tool to call to search for instances
- `suggestedLoaderKeys` — which loaders to use for rendering

### Completeness: partial vs full

Every tool and loader declares a `completeness` level:

- **`"partial"`** — returns a subset of canonical fields. Check `guaranteedFields` to know which are present. Other fields may be absent because they were not fetched, not because they don't exist.
- **`"full"`** — returns all canonical fields. If a field is absent or null, the underlying object genuinely lacks that data.

Search tools are `completeness: "partial"` (they return id, title, handle, image). The `products.get` tool and data loaders are `completeness: "full"`.

### Getting full product details in conversation

When the user asks about variant information, option names, pricing, or other details not available from search, use a session product lookup with the product's GID. This returns a full `Product` object (including `variants`, `options`, `descriptionHtml`) without needing to involve data loaders or page code. Limit this to one product at a time to avoid context explosion.

Example flow:

1. Ask a session to search the catalog for "hoodie" → partial results with id, title, handle, image.
2. User asks "what variants does this have?" → a session product lookup by GID → full product with variants and options.

### Discovery-then-load pattern (for page rendering)

1. Use a search tool to find the right object (returns partial data, including `id` which is the GID).
2. Use `ProductLoader` with `loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}` and `productId={product.id}` from the search result to render full product data on the page.

## Discount Codes

Discount codes are the promo codes a customer types at checkout (e.g. `SUMMER20`). Two read-only tools expose them; both use the Admin API and cover code-based discounts only (not automatic discounts).

- a session discount search — list or search discounts. Call with **no `query`** to list every discount code. Pass a keyword or a field qualifier like `status:active`, `status:expired`, `status:scheduled`, or `title:summer` to filter. Returns partial `DiscountCode` objects (`id`, `codes`, `title`, `status`, `summary`). Never pass placeholders like `"all"` or `"*"` — they are treated as literal search terms.
- a session discount lookup — full detail for one discount. Look it up by its GID (`discountGid`, the `id` from search) **or** by the exact `code` string a customer would enter. Provide exactly one. Returns the codes, status, human-readable `summary`, discount type, start/end dates, usage limit, times used, and whether it applies once per customer.

These are for **discovering and inspecting** discount codes in conversation. Applying a code to a cart is a separate, storefront-side concern handled by `useCart().updateDiscountCodes` in page code — see **Cart Operations**.

## Metafields

Metafields hold a store's custom data — ingredients, size charts, care instructions, spec tables. Two separate things are involved and confusing them is the most common mistake:

- A **definition** is the store-wide declaration that a metafield exists (namespace, key, type). Read with a session metafield-definitions listing.
- A **value** is what a specific product/variant/collection actually stores in it. Read with a session metafield-value read, or inline via a session product lookup / a session collection lookup.

### Always read a value before rendering one

A definition's `type` does not tell you the shape of the data. `rich_text_field` is a JSON document, `list.product_reference` is a JSON array of GID strings, `dimension` is `{"value":5,"unit":"cm"}`. **Never write rendering code from the type name alone** — fetch a real value first and look at it.

```
Ask a session: "List the product metafield definitions on the connected store."
  → { namespace: "custom", key: "ingredients", type: { name: "list.single_line_text_field" }, storefrontAccess: "PUBLIC_READ" }

Ask a session: "Read the custom.ingredients metafield on product
gid://shopify/Product/123."
  → { metafields: [{ namespace: "custom", key: "ingredients", type: "list.single_line_text_field", value: "[\"Cocoa\",\"Sugar\"]" }] }
```

`value` is **always a string**, regardless of type. Parse it according to `type`.

### Which tool to use

| Need                                                      | Tool                                                                                                       |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| What metafields exist on this store?                      | ask a session to list metafield definitions for that `ownerType` — `ownerType` is `product`, `variant`, or `collection` |
| Values for a product, alongside its other data            | ask a session for the product with metafield identifiers                                               |
| Values for a collection, alongside its other data         | ask a session for the collection with metafield identifiers                                         |
| Values for a **variant**, or just metafields on their own | ask a session for metafield values by owner GID                                                        |

Omit `metafieldIdentifiers` / `identifiers` to get every metafield on the owner (capped at 50). Pass them to fetch a targeted subset — do this once you know which keys you want, since it is cheaper and keeps responses small.

Variant metafields are deliberately **not** returned by a session product lookup. A product can have 100 variants, and fetching metafields for all of them in one query exceeds Shopify's query cost limit. Use a session metafield-value read with a variant GID instead.

### `storefrontAccess` — read this before promising the user a metafield on a page

Each definition returns `storefrontAccess`, which is either `"PUBLIC_READ"` or `"NONE"`. It defaults to `NONE` in Shopify.

These tools use Shopify's **Admin** API, but pages render through the **Storefront** API. A definition with `storefrontAccess: "NONE"` is fully readable by these tools and **invisible to a page**. If you build a page around a `NONE` metafield, it will render empty with no error.

If the user wants a `NONE` metafield on a page, tell them it must first be exposed to the Storefront API in Shopify admin (Settings → Custom data → the definition → enable Storefront API access). Do not silently build against it.

### Rendering metafields on a page

`ProductLoader` takes `productMetafieldIdentifiers` and `variantMetafieldIdentifiers`. Pass the **same identifiers, in the same order**, to both `PrefetchedLoaders` and the loader component — the prefetch cache is keyed on the args, so a mismatch silently refetches on the client without metafields.

Product metafields arrive on `product.metafields`; variant metafields on `product.variants[N].metafields`. Both are `{ namespace, key, type, value }`, and `value` is always a string.

```tsx
// app/products/[handle]/page.tsx  (server component)
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";

import { ProductDetail } from "./product-detail";

const PRODUCT_METAFIELDS = [{ namespace: "custom", key: "ingredients" }];

export default async function Page({
  params,
}: {
  params: Promise<{ handle: string }>;
}) {
  const { handle } = await params;
  return (
    <PrefetchedLoaders
      queries={[
        {
          loaderKey: DATA_LOADER_KEYS.SHOPIFY_PRODUCT,
          args: { handle, productMetafieldIdentifiers: PRODUCT_METAFIELDS },
        },
      ]}
    >
      <ProductDetail handle={handle} />
    </PrefetchedLoaders>
  );
}
```

```tsx
// app/products/[handle]/product-detail.tsx  (client component)
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

const PRODUCT_METAFIELDS = [{ namespace: "custom", key: "ingredients" }];

export function ProductDetail({ handle }: { handle: string }) {
  return (
    <ProductLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}
      handle={handle}
      productMetafieldIdentifiers={PRODUCT_METAFIELDS}
    >
      {(product) => {
        const ingredients = product.metafields.find(
          (metafield) =>
            metafield.namespace === "custom" && metafield.key === "ingredients",
        );
        if (!ingredients) {
          return null;
        }
        // `list.single_line_text_field` serializes as a JSON array of strings.
        const items: string[] = JSON.parse(ingredients.value);
        return (
          <ul>
            {items.map((item) => (
              <li key={item}>{item}</li>
            ))}
          </ul>
        );
      }}
    </ProductLoader>
  );
}
```

Rules specific to metafields on pages:

- **Check `storefrontAccess` first.** Loaders read through the Storefront API. A definition with `storefrontAccess: "NONE"` returns nothing here no matter what you pass. Confirm it is `PUBLIC_READ` before building.
- **Always handle the missing case.** A metafield that is defined but unset on a given product is simply absent from the array — that is normal, not an error.
- **Parse according to `type`.** Read a real value with a session metafield-value read first so you know the shape you are parsing.

`CollectionLoader` does **not** accept metafield identifiers yet — collection metafields are readable with a session collection lookup in conversation, but cannot be rendered on a page.

## Metaobjects

Metaobjects are how merchants model **reusable structured content** that is not a product or collection — size charts, ingredient panels, care instructions, FAQ blocks, brand modules. Where a metafield hangs a single custom value off an existing product, a metaobject is a standalone record with its own fields, reusable across many products.

Same definition/value split as metafields, with one hard constraint on top: **Shopify has no query that lists metaobjects across types.** Every read needs a `type` handle, and the only way to learn what types a store has is a session metaobject-definitions listing. Always start there.

```
Ask a session: "List the metaobject definitions on the connected store."
  → { type: "size_chart", displayNameKey: "title", storefrontAccess: "PUBLIC_READ",
      fieldDefinitions: [{ key: "title", type: "single_line_text_field", required: true },
                         { key: "chart", type: "json", required: false }] }

Ask a session: "List the size_chart metaobjects."
  → { metaobjects: [{ handle: "hoodie-sizing", displayName: "Hoodie Sizing",
                      fields: [{ key: "title", type: "single_line_text_field", value: "Hoodie" }] }] }
```

As with metafields, `value` is **always a string** — parse it according to `type`, and read a real entry before writing rendering code.

### Which tool to use

| Need                                              | Tool                                                       |
| ------------------------------------------------- | ------------------------------------------------------------ |
| What metaobject types does this store have?       | ask a session to list metaobject definitions                  |
| All entries of one type                           | ask a session to list metaobjects of that `type`                       |
| One entry, by GID or by type + handle             | ask a session for one metaobject by GID, or by type + handle |

### Resolving a `metaobject_reference` metafield

A metafield whose type is `metaobject_reference` stores a bare metaobject GID as its string value; `list.metaobject_reference` stores a JSON array of them. Neither tells you what is inside. Resolve each GID with ask a session to resolve that GID.

### Two gates before a metaobject can render on a page

These tools use the **Admin** API; pages render through the **Storefront** API. A metaobject readable here can still come back empty at render time, for two separate reasons:

1. **The definition's storefront access.** Same rule as metafields — `storefrontAccess` must be `"PUBLIC_READ"`, and it defaults to `NONE`. Check it before building.
2. **Draft entries.** If the definition has the `publishable` capability, the Storefront API only returns entries that are `ACTIVE`. A newly created entry defaults to `DRAFT`, so an entry visible to these tools may be missing from the page.

Neither failure raises an error — the loader just reports the entry as not found. If either gate is closed, tell the user what to change in Shopify admin (Settings → Custom data → the metaobject definition) rather than silently building against it.

### Rendering metaobjects on a page

`MetaobjectLoader` (loaderKey `shopify_storefront.metaobject`) takes `type` + `handle`, or `id`. `MetaobjectsLoader` (loaderKey `shopify_storefront.metaobjects`) takes a `type` and paginates. Both follow the same prefetch + loader-component pattern as `ProductLoader` — see **Data Loading Pattern**.

Storefront metaobjects have no `displayName` (that is Admin-only), and `fields` only includes keys that have a value — an absent key means null, not an error.

## Usage Rules

- **Always prefer the prefetch pattern** (server component with `PrefetchedLoaders` wrapping a client loader component). Only skip prefetching when the data is entirely driven by client-side interaction with no server-known initial value.
- **Default to ID-based loading** (`productId`) when you know which product to display. Only use handle-based loading for dynamic routes where the product is determined by the URL.
- Use a Replo session for catalog discovery in conversation. Admin searches return partial Product/Collection objects with `id` (GID), `title`, `handle`, and `featuredImage`.
- Use a session product lookup when you need full product details (variants, options, pricing) for conversational answers — not for page rendering.
- Never write code that renders a metafield without first reading an actual value with a session metafield-value read — the type name does not tell you the shape of the data. See **Metafields**.
- For structured content that is not a product or collection (size charts, ingredient lists, FAQ blocks), check a session metaobject-definitions listing before telling the user the data is not available. See **Metaobjects**.
- Use Storefront loaders for rendering page data (`ProductLoader`, `CollectionLoader`, `CollectionProductsLoader` with the appropriate `loaderKey`). These return full objects.
- For cart operations, use `product.variants[N].id` as the merchandiseId — this is the same Shopify variant GID returned by loaders.
- For dynamic pages, read route params with `useParams()` or the page `params` prop and pass them into loader component props.
- Treat `availableForSale` as availability; do not infer inventory quantities. Product-level `availableForSale` is `true` if **any** variant is purchasable, so use the per-variant `availableForSale` on `product.variants[]` (returned by both `ProductLoader` and `CollectionProductsLoader`) for accurate sold-out UI.

## Cart Operations

### Architecture

Cart operations go through the **dependency-resolver** (server-to-server). The scaffold never calls Shopify directly for cart mutations. This means:

- No Shopify credentials are needed in the scaffold for cart.
- Cart hooks call server actions, which call the resolver, which calls Shopify.
- Cart state is persisted via a cookie (`replo_cart_id`) containing only the Shopify cart GID.

### Hooks

Import cart hooks from `@replohq/sdk/cart/hooks/`:

- **`useAddToCart`** — adds items to the cart using merchandise IDs from loader output.
- **`useBuyNow`** — creates a cart and redirects directly to checkout (skips the cart UI).
- **`useCart`** — read-only cart UI state: `itemsCount`, `isCartOpen`, `openCart`, `closeCart`, `updateDiscountCodes`.

### Add to Cart

Use `useAddToCart` with the variant's `id` from a data loader. The variant `id` is the Shopify variant GID — pass it directly as `merchandiseId`. The hook auto-resolves product display data (title, price, image) from the React Query cache for optimistic cart UI — no need to pass it manually. Cache resolution works for both single-product loaders (`ProductLoader`) and product-list loaders (`CollectionProductsLoader`, `RebuyRecommendationsLoader`).

```tsx
"use client";

import { useAddToCart } from "@replohq/sdk/cart/hooks/use-add-to-cart";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

function ProductCard({ productId }: { productId: string }) {
  const { addToCart, isAdding } = useAddToCart();

  return (
    <ProductLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}
      productId={productId}
      fallback={<div>Loading…</div>}
    >
      {(product) => {
        const firstVariant = product.variants[0];
        return (
          <div>
            <h2>{product.title}</h2>
            <button
              disabled={!firstVariant?.availableForSale || isAdding}
              onClick={() => {
                if (firstVariant) {
                  addToCart([{ merchandiseId: firstVariant.id, quantity: 1 }]);
                }
              }}
            >
              {isAdding ? "Adding…" : "Add to Cart"}
            </button>
          </div>
        );
      }}
    </ProductLoader>
  );
}
```

### Add to Cart from a Collection Grid

When wiring an Add to Cart button inside a `CollectionProductsLoader` grid, see [add-to-cart-from-collection-grid.md](references/add-to-cart-from-collection-grid.md). Key rule: pass the variant GID (`product.variants[N].id`), never `product.id`.

### Buy Now

Use `useBuyNow` for express checkout. Pass variant IDs the same way.

```tsx
"use client";

import { useBuyNow } from "@replohq/sdk/cart/hooks/use-buy-now";

function BuyNowButton({ variantId }: { variantId: string }) {
  const { buyNow } = useBuyNow();

  return (
    <button
      onClick={() => {
        buyNow([{ variantId, quantity: 1 }]);
      }}
    >
      Buy Now
    </button>
  );
}
```

`useBuyNow` creates a Shopify cart and redirects to **Shopify Checkout**. Wallet options (Shop Pay, Apple Pay, Google Pay) can appear **inside** that hosted checkout. `useBuyNow` is **not** an on-page "Buy with Shop Pay" button in itself.

### Shop Pay (on-page Buy with Shop Pay)

When the user asks for Shop Pay / Buy with Shop Pay on the page (Shopify-checkout projects only), see [shop-pay-button.md](references/shop-pay-button.md). Do not hardcode it onto every PDP by default. Do not confuse it with `useBuyNow`.

### Cart UI

**Do not assume a slide-out cart component exists — verify it, and build it if it's missing.** Cart _context_ is always available (any template wrapped in `ReploProvider` gives every page `useCart()` / `useAddToCart` / `useBuyNow`), but the visual cart is an ordinary component of the site — in `components/cart/`, so it is not a surfaced Site Component — that only some templates ship. The blank starter includes it; branded and other templates may not. Before wiring any "Add to Cart" button or cart icon to `openCart()`, confirm the component actually exists.

**1. Check for the cart component and its render:**

- Look for the component file at `@/components/cart/slideout-cart.tsx` (conventional location).
- Confirm it is rendered once in the root layout — a `<SlideoutCart />` inside `ReploProvider` in `app/layout.tsx`.

**2. If it exists,** use it as-is via `openCart()` (see below). Do not render it again in individual pages.

**3. If it is missing, build it and wire it into the layout yourself:**

- Create a client component (`"use client"`) at `@/components/cart/slideout-cart.tsx` that reads UI state from `useCart()` (`isCartOpen`, `openCart`, `closeCart`, `itemsCount`) plus the cart lines, and renders a right-side drawer with an overlay backdrop.
- Render it exactly once inside `ReploProvider` in `app/layout.tsx` (e.g. right after `{children}`), never per-page.
- If the layout has no `ReploProvider` at all (some plain templates), the cart hooks won't work until that runtime layer exists — add `ReploProvider` from `@replohq/sdk/providers/replo-provider` (wrapping `SandboxRuntime` from `@replohq/sandbox-runtime`) around the app first.
- Use the cart display utilities (`getCartLinePricing`, `getCartLineBundleInfo`, `getCartLineSpecialInfo`, `getCartLineSubscriptionInfo`) from `@replohq/sdk/cart/utils/variant-to-cart-line` for line-item pricing and badges.

A complete slide-out cart handles:

- Line item display with images, options, and pricing
- Quantity controls (increment/decrement/remove)
- Discount code application and removal
- Bundle and subscription badge display
- Subtotal, discounts, and total calculation
- Checkout link

To open the cart from any component (e.g., an "Add to Cart" button or a cart icon in a header):

```tsx
"use client";

import { useCart } from "@replohq/sdk/cart/cart-provider";

function CartButton() {
  const { itemsCount, openCart } = useCart();
  return <button onClick={openCart}>Cart ({itemsCount})</button>;
}
```

The slide-out cart opens from the right side of the viewport with an overlay backdrop. Clicking outside the cart or pressing the X button closes it.

**Customizing the cart UI:** The `SlideoutCart` component is a starting point. You can freely modify its styling (it uses standard Tailwind classes), add sections, or restructure it. The cart display utilities used by the component (`getCartLinePricing`, `getCartLineBundleInfo`, `getCartLineSpecialInfo`, `getCartLineSubscriptionInfo`) are imported from `@replohq/sdk/cart/utils/variant-to-cart-line` and can be used in any custom cart UI.

### Cart Rules

- **Never call resolver cart endpoints directly from client code.** Always use the cart hooks which go through server actions.
- The `merchandiseId` for add-to-cart is `product.variants[N].id` — the Shopify variant GID from loader output. No ID conversion needed.
- `useCart()` provides UI state only. For mutations use `useAddToCart` (add items) or `useBuyNow` (express checkout).
- On-page **Buy with Shop Pay** is separate from cart hooks — see [shop-pay-button.md](references/shop-pay-button.md). Do not confuse it with `useBuyNow`.
- `ReploProvider` in the root layout provides cart context to every page. Cart _hooks_ work automatically wherever it is present; the cart _UI component_ does not — verify it exists (and build it if not) before wiring buttons to `openCart()`.
- The `SlideoutCart` (when present, or once you've added it) is rendered once in the root layout — do not render it again in individual pages.
