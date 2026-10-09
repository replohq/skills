# Intelligems code audit

Run this against the site's source, installed package types, configuration and local dev server before publishing. It checks whether the integration is implemented correctly; it does not certify live assignment, Shopify orders or Intelligems reporting. Run the [live experiment audit](live-experiment-audit.md) on the published page when requested and before declaring the experiment ready for traffic.

For every applicable item, record **passed**, **failed**, **blocked**, or **not applicable**, with file/line evidence and a short explanation. Include relevant code snippets and local verification results. Never include token values. Fix failed code checks and rerun them; identify external configuration that still needs live verification.

## 1. Provider and connection

Read the [Next.js App Router integration](https://headless.intelligems.io/integration-guides/next.js-or-app-router) and [provider props](https://headless.intelligems.io/reference/providers/provider-props).

- [ ] `@intelligems/headless` >= 1.2.19 is installed in this site. For Replo/Shopify shared-domain journeys, follow the [required package patch](package-patch.md): pin 1.2.19, commit the pnpm patch and registration, verify the bundle hash, and set `cookieDomain` before initialization, or verify a vendor-fixed release. A version minimum alone does not prove the bug is fixed. Verify every import against the installed package.
- [ ] One `IntelligemsNextClientsideAppDirectoryProvider` owns the experiment subtree. No standard Intelligems script is also injected through Scripts & Pixels, GTM or page code. Replo hooks are under `ReploProvider`.
- [ ] `ShopifyIntegrationDetailsLoader`, with `DATA_LOADER_KEYS.SHOPIFY_INTEGRATION_DETAILS`, supplies `storefrontAccessToken` to `storefrontApiToken`. Its render prop also returns `shopifyDomain`; confirm the intended store. Optional `PrefetchedLoaders` uses the same key and `args: {}`. Verify the installed Replo SDK exports both. No manually copied token or Admin/External API credential in page code.
- [ ] Organization, experience and variation IDs come from the connected account or dashboard, not names or guessed IDs. Record which external configuration remains unverified.
- [ ] Country and currency come from the site's selected-market cookie resolver and configured market entries, with a valid merchant default. Shopify loader/prefetch country, `ReploProvider` markets, provider `activeCurrencyCode` and tracking props agree. Do not derive market from language or require a product/cart to obtain currency.
- [ ] If multiple markets are supported, changing the cookie refreshes server props and reconstructs the Intelligems client as needed by the installed version (the 1.2.19 recipe keys the provider by country and currency).

## 2. Assignment, tracking and regular cart

Read [variation hooks](https://headless.intelligems.io/reference/hooks/variation-hooks), [track hooks](https://headless.intelligems.io/reference/hooks/track-hooks) and [cart hooks](https://headless.intelligems.io/reference/hooks/cart-and-checkout-hooks).

- [ ] Find **every `useIgTrack` call**. Each receives the current `cartId` from Replo's `useCart()` as `cartOrCheckoutToken`. No hardcoded ID, checkout URL, independently parsed cookie or snapshot that stops updating when the cart changes. A null ID before cart creation is valid; the hook must receive the new ID on rerender.
- [ ] There is one native page-view tracker owner, not one per product or tested section. `useIgTrack` already includes `useIgCart`, custom-event setup and GA4 variation tracking; do not add duplicate trackers for those duties. Explicit `useIgCart(cartId)` is useful when its returned line-property wrapper is needed.
- [ ] All assignments required to render the measured content are ready before mounting that tracker. Use a child component to gate mounting, not conditional hook calls. A resolved control is measured; a loading/error fallback is not a treatment exposure. `useIgTrack` is not an experiment-active or consent gate on its own.
- [ ] Loading leaves the rest of the page usable; thrown errors show usable default content through an error boundary. No custom timeout is required. A recovered assignment and the measured content must agree.
- [ ] Normal cart mutations still use Replo's cart hooks. Audit the checkout action for asynchronous attribute writes: hook readiness is not a write-completion guarantee. A fixed delay is not a persistence barrier; unresolved races must be carried into the live audit.

Tracker example, mounted only inside the ready, consent-allowed experiment subtree:

```tsx
import { useIgTrack } from "@intelligems/headless/next-clientside-app-directory";
import { useCart } from "@replohq/sdk/cart/cart-provider";

function ExperimentTracker({ market }: {
  market: { country: string; currencyCode: string };
}) {
  const { cartId } = useCart();
  useIgTrack({
    cartOrCheckoutToken: cartId,
    country: market.country,
    currency: market.currencyCode,
  });
  return null;
}
```

## 3. Prices and checkout pricing

Read [price hooks (`useIgPrices`)](https://headless.intelligems.io/reference/hooks/price-hooks), [price components](https://headless.intelligems.io/reference/components/price-components), [updating page prices](https://headless.intelligems.io/usage/update-prices-on-page), and [Shopify Functions cart integration](https://headless.intelligems.io/usage/update-atc-events/shopify-functions).

- [ ] Inventory every price surface: product cards, selected variant, sticky purchase UI, compare-at price, savings, bundles and cart. Experiment-sensitive product prices use `useIgPrices` or `IgPrice`/`IgCompareAtPrice`; changing one text node is insufficient.
- [ ] Inputs follow the selected product/variant and market. Supply original Shopify prices for fallback, in the units expected by the installed SDK; verify decimal-string/minor-unit conversion rather than guessing or multiplying by 100 everywhere. Format returned amounts with their currency and preserve zero prices with nullish checks.
- [ ] Respect `isReady`; do not use `isIgPrice` as a readiness flag. A ready original price can be valid when no price experiment applies. Check the installed hook types for optional/null prices and any `duplicateProduct` result.
- [ ] A price test's Shopify checkout mechanism is configured. For Functions-based pricing, maintain the experiment cart attributes. If the store uses duplicate-product pricing, follow its supported mapping, including returned variant identity, through purchase and subsequent product loading; block unsupported mappings rather than charging the original variant silently.
- [ ] Selected quantity and selling plan reach the purchase call. `useIgPrices` does **not** document a `sellingPlanId` argument: do not invent one. Subscription/allocation pricing and discounts need an explicitly supported integration and separate checkout verification.
- [ ] Cart totals use Shopify's returned amounts; never simulate checkout pricing by overriding only displayed cart totals. Mark price tests not applicable if the site has none, and blocked if the required pricing integration is unavailable.

Price-hook excerpt inside a client component under the provider; the selected variant must come from the same market's product data. Verify the installed signature before adapting it:

```tsx
const prices = useIgPrices({
  productId: product.id,
  variantId: selectedVariant.id,
  originalPrice: selectedVariant.price.amount,
  originalCompareAtPrice: selectedVariant.compareAtPrice?.amount,
  currencyCode: selectedVariant.price.currencyCode,
});
if (!prices.isReady) return <PricePlaceholder />;
const amount = prices.igPrice?.value ?? Number(selectedVariant.price.amount);
const currencyCode = prices.igPrice?.currencyCode ?? selectedVariant.price.currencyCode;
```

Import `useIgPrices` from the installed headless entry point used by the provider. `PricePlaceholder` is site-defined. Apply the same hook result to compare-at/savings rendering; it is not proof that Shopify will charge this amount.

## 4. Buy Now and shipping line properties

Read [cart and checkout hooks](https://headless.intelligems.io/reference/hooks/cart-and-checkout-hooks) and [updating add-to-cart events](https://headless.intelligems.io/usage/update-atc-events).

- [ ] Buy Now uses `useIgCartAttributes()` and passes its exact attributes to `useBuyNow().buyNow(variants, { attributes })` at cart creation. It creates a separate cart; the regular cart's tracked ID cannot attribute it. Verify the installed Replo SDK supports this option.
- [ ] Gate attributed checkout on readiness and complete string-valued attributes. Handle nullable SDK types without casts or silently discarding missing values. Preserve the normal consent-declined checkout path without mounting Intelligems just to obtain attributes.
- [ ] Buttons retain pending/error handling and duplicate-click prevention. Checkout links/permalinks that bypass cart creation need to be replaced with the supported action for attributed Buy Now; query parameters alone do not implement this flow.
- [ ] For shipping tests, obtain `wrapCustomAttributes` from `useIgCart(cartId)`, with `cartId` from `useCart()`. Wrap the selected product/variant, `subscribeAndSave` state and existing properties. Pass the result through each line's Replo `properties` field for both `useAddToCart` and `useBuyNow`. Cart-level attributes do not replace line properties.

Buy Now handler excerpt; hooks belong at component top level under both providers. Include the wrapping step only when the experiment requires line properties:

```tsx
const { cartId } = useCart();
const { isReady, attributes } = useIgCartAttributes();
const { isReady: cartReady, wrapCustomAttributes } = useIgCart(cartId);
const { buyNow } = useBuyNow();

async function handleBuyNow() {
  const completeAttributes = attributes.filter(
    (attribute): attribute is { key: string; value: string } =>
      typeof attribute.value === "string",
  );
  if (!isReady || !cartReady || completeAttributes.length !== attributes.length) return;
  await buyNow([{
    variantId: selectedVariant.id,
    quantity,
    sellingPlanId: selectedSellingPlanId,
    properties: wrapCustomAttributes({
      productId: product.id,
      variantId: selectedVariant.id,
      subscribeAndSave: Boolean(selectedSellingPlanId),
      customAttributes: existingLineProperties,
    }),
  }], { attributes: completeAttributes });
}
```

The selected product, variant, quantity, plan and existing line properties are site values. Wire disabled/pending state and handle the Buy Now result using the site's purchase UI. Keep normal add-to-cart on `useAddToCart`, supplying the same wrapped line properties where required.

## 5. Redirects, consent and goals

- [ ] Redirect code uses the real configured experience and destination, preserves required query parameters, prevents loops and excludes non-origin routes. Check initial load **and Next.js client navigation**; a persistent provider may need reevaluation/remounting for the new route. Do not assume mounting headless implements redirects automatically.
- [ ] Shared identity is configured for supported merchant domains; no claim that unrelated domains or `replosites.com` share cookies. Record any pinned cookie-domain patch for live migration tests. No invented `recordExposure` or server assignment API. Any server proposal must defer unknown/browser-only eligibility to the client, not assign control for unknown state.
- [ ] If the site uses consent, provider, tracker and custom-event handlers obey its CMP before initialization and after revocation. Returning users and hydration are covered; no replay of dropped goals. If policy requires deleting existing attribution, implement that separately from unmounting.
- [ ] [Custom events](https://headless.intelligems.io/examples/custom-events) map deliberate successful actions to the configured event identifiers, with consistent cohort semantics and duplicate protection. No blanket click tracking, personal quiz answers or duplicated native commerce events. Telemetry checks apply even when the site has no consent UI.
- [ ] For offers, consult [offer hooks](https://headless.intelligems.io/reference/hooks/offer-hooks); audit tier eligibility, discount stacking and any gift add/remove behavior. Hooks exposing an offer do not implement its cart mutations automatically.

Conclude with code changes made, unresolved implementation issues and the applicable [live experiment audit](live-experiment-audit.md) cases. Label the result **code audit passed** only; reserve launch verification for published evidence.
