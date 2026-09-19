---
name: product-display
title: Display Products
summary: Load and display product data with the right architecture.
description: "REQUIRED when building any page or component that displays product data. Contains product display requirements, pricing guidelines, and description rendering rules. Applies to both Shopify and Replo products."
tools: find_products, get_product
---

# Product Display

Products are loaded via `ProductLoader` (from `@replohq/sdk/loaders/product-loader`) using a render-prop pattern. The product data shape is the same regardless of source (Shopify or Replo) — see the **shopify** skill for source-specific data loading and cart patterns.

**CRITICAL:** Never hardcode live product data. Use `ProductLoader` for a single product, or the source-specific collection/recommendation loader for a product list. Reuse the real product objects those loaders return; do not add a per-item fetch when the required data is already loaded.

## Main Product Display

When building the **main product display** on a page (i.e., the primary product the page is focused on), the following guidelines **MUST** be followed unless the user explicitly specifies otherwise:

**Required Elements (accessed dynamically from the `product` parameter in the `ProductLoader` render prop):**

- **Title** — Display `product.title`
- **Price** — Convert the loader's major-unit amount with `fromMajorUnitsToMinorUnits({ amount: variant.price.amount, currencyCode: variant.price.currencyCode })` from `@replohq/sdk/money`, then pass that integer to `useFormattedPrice(minorUnits, variant.price.currencyCode)` from `@replohq/sdk/hooks/use-formatted-price`
- **Description** — Use `product.descriptionHtml` with `dangerouslySetInnerHTML={{ __html: product.descriptionHtml }}` to render the rich HTML description
- **Image** — Display `product.featuredImage` (and variant images when available)
- **Quantity Selector** — Allow users to select quantity
- **Add to Cart Button** — Primary CTA using `useAddToCart` from `@replohq/sdk/cart/hooks/use-add-to-cart`
- **Buy Now Button** — Secondary CTA using `useBuyNow` from `@replohq/sdk/cart/hooks/use-buy-now`

**Conditional Requirements:**

- **Multiple Images:** If `product.images` has more than one entry, you **SHOULD** include an image gallery or carousel. If variants have distinct images, swap the displayed image on variant selection
- **Product Options:** If `product.options` has entries, you **MUST** include option selectors for all available options (size, color, etc.) that update the selected variant
- **Availability:** Check `variant.availableForSale` — disable Add to Cart/Buy Now for unavailable variants

## Upsell/Collection/Recommended Product Display

When building **upsell products, recommended products, or collection items** (i.e., not the main product on the page), the requirements are more flexible:

**Minimum Required Elements:**

- **Title** — Display `product.title`
- **Price** — Show the current price
- **Image** — Display `product.featuredImage` (carousel not required)
- **CTA** — Add to cart button

**Optional Elements (include if space and context permit):**

- Description (can be truncated)

**Note:** For upsells and recommendations, prioritize clean, scannable layouts over comprehensive information. The goal is to entice the customer to learn more or quickly add to cart.

## Pricing

- Format all prices using `useFormattedPrice()` from `@replohq/sdk/hooks/use-formatted-price` — never format manually
- Variant prices are decimal major units (`"48.00"` means $48), while `useFormattedPrice` accepts integer minor units (`4800` means $48). Convert with `fromMajorUnitsToMinorUnits` from `@replohq/sdk/money`; do not multiply by 100 because currency exponents vary
- `variant.compareAtPrice` is available on `FullVariant` as `{ amount: string, currencyCode: string } | null`. When non-null, display a strikethrough "was" price alongside the current price to show the discount
- Cart-level pricing utilities (e.g., `getCartLinePricing` from `@replohq/sdk/cart/utils/variant-to-cart-line`) also support compare-at pricing for cart line items
- Only show per-unit breakdowns when there is an explicit unit basis (servings, capsules, packs, cases). If unknown, do **NOT** display per-unit pricing

## Selling Plans (Subscriptions)

Selling plan data is available directly on the product from `ProductLoader`:

- **`product.sellingPlanGroups`** — array of `SellingPlanGroup`, each containing `appId`, `options`, and `sellingPlans` (with `id`, `name`, `description`, `optionValues`, and `priceAdjustments`)
- **`variant.sellingPlanIds`** — array of selling plan GIDs that this specific variant is eligible for. Use this to filter `product.sellingPlanGroups` so the UI only shows plans available for the selected variant

**To build a subscription product UI:**

1. Render a plan selector using `product.sellingPlanGroups` — filter each group's `sellingPlans` to those whose `id` appears in the selected `variant.sellingPlanIds`
2. Pass `sellingPlanId` when calling `addToCart` or `buyNow`:
   ```tsx
   addToCart([{ merchandiseId: variant.id, quantity: 1, sellingPlanId: selectedPlanId }]);
   ```
3. Products with no selling plans will have `sellingPlanGroups: []` and `variant.sellingPlanIds: []`

**Cart-level selling plan utilities:**

- `CartLineMerchandise.product.sellingPlanGroups` is also populated in the optimistic cart for immediate UI feedback
- Use `getCartLinePricing(cartLine)` from `@replohq/sdk/cart/utils/variant-to-cart-line` to get `sellingPlanAdjustedPrice` and `finalPrice`
- Use `getCartLineSubscriptionInfo(cartLine)` to get `isSubscription`, `frequency`, and `sellingPlan` details
- Use `createOptimisticSellingPlanAllocation()` to build optimistic UI updates when adding items with selling plans

## Product Description Fields

- **`descriptionHtml`** — Rich HTML content. **ALWAYS use this for rendering descriptions on the page** via `dangerouslySetInnerHTML={{ __html: product.descriptionHtml }}`. This preserves formatting like bold, lists, links, etc.
- **`description`** — Plain-text version of the description (HTML stripped by Shopify). Use this for SEO meta tags, schema.org structured data, `aria-label` attributes, and anywhere HTML rendering is not appropriate.

**WARNING:** Never render `descriptionHtml` outside of `dangerouslySetInnerHTML` — it contains HTML that will show as raw markup.

## Product Images

- **`product.featuredImage`** — nullable hero image. Check it before reading its URL or dimensions; use an available product/variant image or omit it when none exists
- **`product.images`** — full array of all product images. Use this for image galleries and carousels on PDPs. Each image has `url`, `altText`, `width`, and `height`
- Variant-specific images are available via `variant.image` — use these to swap the displayed image when the user changes variant selection

## Merchandising Fields

- **`product.vendor`** — brand/vendor name (`string | null`). Use for vendor badges, `by <vendor>` attribution, or filtering
- **`product.productType`** — Shopify product type (`string | null`). Use for categorization, breadcrumbs, or filtering
- **`product.tags`** — array of tag strings. Use for badges (e.g., "Bestseller", "New"), conditional rendering, or filtering

## Missing Product IDs or No Products

Discover a real product using the **shopify** skill or `find_products` for Replo-managed products, then wire the loader and matching prefetch arguments. Never invent an ID such as `"1"`.

If the catalog is empty, show an explicit no-product state and keep purchase controls unavailable until a real product is bound. Editor-only sample data for unfilled dynamic-route arguments such as `[slug]` is preview scaffolding, not a published-site fallback. Verify the concrete route with a real product before claiming product display or checkout works.
