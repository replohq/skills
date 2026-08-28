# Shop Pay via hydrogen-react `ShopPayButton`

On-page **Buy with Shop Pay** for Shopify-checkout agent storefronts. Complements cart / Buy Now (which land in hosted Shopify Checkout).

## When to use

- User asks for Shop Pay, Buy with Shop Pay, or one-click Shop Pay on a product page or cart.
- Project checkout provider is Shopify (`CHECKOUT_PROVIDER === "shopify"` in `wrangler.jsonc` — read it, do not edit it).

Skip when checkout is Stripe / none, or when the user only wants a normal Buy Now → Shopify Checkout redirect (`useBuyNow`). Do **not** hardcode Shop Pay onto Stripe checkout pages.

## Gate first

1. Confirm `CHECKOUT_PROVIDER` is `"shopify"`. If it is `"stripe"` or `"none"`, use the project's existing checkout path instead.
2. Merchant prerequisites (outside your control): Shopify Payments + Shop Pay enabled on the store. If the button fails to open checkout, tell the user to verify those settings in Shopify admin — do not invent a payment-request / Wallet API backend.
3. Resolve the store domain **before** writing the component (see **Store domain** below). Bake it into the client component as a string constant. Do **not** wrap the app in Hydrogen `ShopifyProvider` just for this button — that provider wants a Storefront token and API version you should not plumb through the scaffold for Shop Pay alone.

## Install

In the site repo:
```bash
pnpm add @shopify/hydrogen-react
```

## Verified API notes (`@shopify/hydrogen-react` 2026.4.x)

Inspected from package source + [ShopPayButton docs](https://shopify.dev/docs/api/hydrogen-react/latest/components/shoppaybutton):

| Concern | Behavior |
| --- | --- |
| Import | `import { ShopPayButton } from "@shopify/hydrogen-react"` |
| Provider | Optional. Without `ShopifyProvider`, **`storeDomain` prop is required**. Prefer props-only — do not add `ShopifyProvider` solely for this button. |
| `storeDomain` | Hostname like `your-store.myshopify.com` (also accepts `https://…`). Passed through to `<shop-pay-button store-url>`. |
| Variant IDs | **GIDs only** (`gid://shopify/ProductVariant/…`). Bare numerics throw: must use `["gid://shopify/ProductVariant/1"]` form. Component strips to numeric IDs for the web component. |
| Quantity | `variantIds` ⇒ qty 1 each. Else `variantIdsAndQuantities: [{ id, quantity }]`. Never both. |
| `channel` | Optional `"headless"` \| `"hydrogen"` attribution. **Omit by default.** Verified on Shopify: with `channel` the cart permalink returns 400 (`Parameter Missing or Invalid: channel`); without `channel` it 302s to `shop.app` Shop Pay checkout. |
| Client / SSR | Uses `useLoadScript` for `https://cdn.shopify.com/shopifycloud/shop-js/v1.0/client.js`. Must be `"use client"`. SSR output is an empty wrapper `div` until the script status is `"done"`. |
| Styling | `className` on the wrapper; `width` sets `--shop-pay-button-width` (e.g. `"100%"`). |
| CSP | Script host is `cdn.shopify.com`. Agent published pages do not ship a page-level CSP that blocks it (the `script-src 'none'` in scaffold `next.config` applies only to Next Image SVG handling). |

## Store domain

1. Ask a Replo session for the store's `myshopifyDomain`, **or**
2. Read `vars.SHOPIFY_URL` from `wrangler.jsonc` (publish writes it when Shopify is connected; do not edit that file).

Bake the hostname into the client component. `getEnv()` from `@replohq/sdk` does **not** expose `SHOPIFY_URL` to typed app code.

## Placement

- PDP: under Add to Cart / beside Buy Now; bind to the selected variant GID + qty from existing UI state.
- Cart (optional): map line items to `variantIdsAndQuantities` using each line's merchandise variant GID.
- Gate render: only mount when you have a non-empty `storeDomain` and at least one variant GID. Sold-out / missing selection → omit the button. Do not catch render throws as a substitute for valid props.

## Minimal PDP example

```tsx
"use client";

import { ShopPayButton } from "@shopify/hydrogen-react";

// The store's myshopifyDomain.
const STORE_DOMAIN = "your-store.myshopify.com";

export function BuyWithShopPayButton({
  variantGid,
  quantity = 1,
}: {
  variantGid: string | null | undefined;
  quantity?: number;
}) {
  if (!variantGid || !STORE_DOMAIN || quantity < 1) {
    return null;
  }

  return (
    <ShopPayButton
      storeDomain={STORE_DOMAIN}
      variantIdsAndQuantities={[{ id: variantGid, quantity }]}
      width="100%"
      className="w-full"
    />
  );
}
```

## Failure modes

- Shop Pay / Shopify Payments disabled → button may render but checkout fails; tell the merchant to enable them in Shopify admin.
- Wrong `storeDomain` → checkout targets the wrong shop.
- Draft / unavailable variants → omit the button; use per-variant `availableForSale`.
- Invalid props (missing domain, non-GID, both variant props) → **throws at render** — gate with null checks instead of try/catch around JSX.

## Out of scope

- On-page Apple Pay / Google Pay for Shopify checkout projects (those wallets still appear inside Shopify Checkout after cart / Buy Now redirect; do not promise theme-parity dynamic wallet button rotation). Shopify does not support these third-party one-click checkouts on headless pages.
- Stripe Express Checkout Element (Replo/Stripe checkout projects)
