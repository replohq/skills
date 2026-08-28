---
name: product-display
title: Display Products
summary: Load and display product data with the right architecture.
description: "REQUIRED when building any page or component that displays product data. Contains product display requirements, pricing guidelines, and description rendering rules. Applies to both Shopify and Replo products."
tools: find_products, get_product
---

# Product Display

Products are loaded via `ProductLoader` (from `@replohq/sdk/loaders/product-loader`) using a render-prop pattern. The product data shape is the same regardless of source (Shopify or Replo) — see the **shopify** skill for source-specific data loading and cart patterns.

**CRITICAL:** NEVER hardcode product data. Always use `ProductLoader` to access product data dynamically.

## Main Product Display

When building the **main product display** on a page (i.e., the primary product the page is focused on), the following guidelines **MUST** be followed unless the user explicitly specifies otherwise:

**Required Elements (accessed dynamically from the `product` parameter in the `ProductLoader` render prop):**

- **Title** — Display `product.title`
- **Price** — Show the current price using `useFormattedPrice(Number(variant.price.amount), variant.price.currencyCode)` from `@replohq/sdk/hooks/use-formatted-price`
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
- Variant prices are `{ amount: string, currencyCode: string }` — convert `amount` to a number before passing to `useFormattedPrice`
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

- **`product.featuredImage`** — single hero image, always available for quick display
- **`product.images`** — full array of all product images. Use this for image galleries and carousels on PDPs. Each image has `url`, `altText`, `width`, and `height`
- Variant-specific images are available via `variant.image` — use these to swap the displayed image when the user changes variant selection

## Merchandising Fields

- **`product.vendor`** — brand/vendor name (`string | null`). Use for vendor badges, `by <vendor>` attribution, or filtering
- **`product.productType`** — Shopify product type (`string | null`). Use for categorization, breadcrumbs, or filtering
- **`product.tags`** — array of tag strings. Use for badges (e.g., "Bestseller", "New"), conditional rendering, or filtering

## Missing Product IDs or No Products

Always use `ProductLoader` and the prefetch pattern, even when the user hasn't specified a product or has no products. See the **shopify** skill for how to search for products when you don't have a specific product ID, and the `find_products` tool for Replo-managed products.

If the user has no products, use the mock product ID `"1"`. The product data system automatically falls back to a predefined mock product when needed.
