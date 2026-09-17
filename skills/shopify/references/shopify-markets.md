# Shopify Markets (country / currency)

Shopify **Markets** control which currency, prices, and (when configured) market-specific catalog a Storefront request sees. That is a different concept from **translations**:

| Concept | Where to pass it | Storefront field | What it changes |
| --- | --- | --- | --- |
| Translation / language | Loader `language` | `@inContext(language:)` | Product titles, descriptions, options, metafield text |
| Market / country (catalog) | Loader `country` | `@inContext(country:)` | Currency code, money amounts, market-aware catalog |
| Market / country (cart) | `ReploProvider markets.shopify.country` | `buyerIdentity.countryCode` | Cart line prices and checkout currency |

Pass `language` and `country` independently on `ProductLoader`, `CollectionLoader`, and `CollectionProductsLoader`. Language alone never changes currency; country alone never translates catalog text. For `language`, see [translating-shopify-content.md](translating-shopify-content.md).

**Catalog `country` is not cart market.** Product cards can show EUR while the cart drawer and Shopify checkout stay in the shop default (e.g. GBP) unless the same country also reaches `ReploProvider` as `markets={{ shopify: { country } }}`. Always do both when the page is in a market.

## When to use this

- The merchant uses Shopify Markets (or international pricing) and wants prices in a market's currency (e.g. EUR on an `en-eu` URL while the shop default is GBP).
- A page or route represents a market (URL prefix, host, or explicit market switcher), not merely a translated dictionary.
- The user asks for currency, Markets, regional pricing, cart/checkout currency, or "show € / $ / £ for this locale."
- Product cards already show a market currency but the cart drawer or checkout is still in the shop default.

Do **not** treat locale routing or dictionary work as Markets work. Site i18n (`reploLocaleRouting`, `dictionaries/*.json`) picks which language copy to show; Markets/`country` picks which money amounts Storefront returns.

## Pass `country` to the loaders

`country` is a Shopify Storefront [`CountryCode`](https://shopify.dev/docs/api/storefront/latest/enums/CountryCode) (`FR`, `GB`, `US`, …) — ISO 3166-1 alpha-2. Import `StorefrontCountryCode` from `@replohq/sdk/loaders/markets` for type-level validation. There is no `EU` country code.

Map the page's market signal to a real Markets country. For BCP-47 tags with a region (`en-CA`, `fr-FR`), the region subtag usually works. For custom market prefixes without a region (`en-eu`), pick a representative country that is in the merchant's euro (or other) Market — e.g. `FR` or `DE` — after confirming Markets is configured for it:

```tsx
// app/[lang]/products/[handle]/ProductDetail.tsx
"use client";

import type { StorefrontCountryCode } from "@replohq/sdk/loaders/markets";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

/** Map a site market signal to a Storefront CountryCode. */
function toShopifyCountry(lang: string): StorefrontCountryCode | undefined {
  const locale = new Intl.Locale(lang);
  if (locale.region) {
    return locale.region.toUpperCase() as StorefrontCountryCode;
  }
  // Custom market prefixes without a region subtag — e.g. en-eu for EUR Markets.
  const marketCountries: Record<string, StorefrontCountryCode> = {
    "en-eu": "FR",
    eu: "FR",
  };
  return marketCountries[lang.toLowerCase()];
}

export function ProductDetail({
  lang,
  handle,
}: {
  lang: string;
  handle: string;
}) {
  const country = toShopifyCountry(lang);

  return (
    <ProductLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}
      handle={handle}
      country={country}
      fallback={<div>Loading…</div>}
    >
      {(product) => (
        <p>
          {product.variants[0]?.price.amount}{" "}
          {product.variants[0]?.price.currencyCode}
        </p>
      )}
    </ProductLoader>
  );
}
```

Wire the same `country` into `PrefetchedLoaders` so the React Query key matches:

```tsx
<PrefetchedLoaders
  queries={[
    {
      loaderKey: DATA_LOADER_KEYS.SHOPIFY_PRODUCT,
      args: { handle, country },
    },
  ]}
>
  <ProductDetail lang={lang} handle={handle} />
</PrefetchedLoaders>
```

When the same page also needs translated catalog text, pass `language` and `country` together in both the loader props and the prefetch args — still as two separate concepts, not one "locale" blob.

`CollectionLoader` and `CollectionProductsLoader` accept the same optional `country` field. Pass it on collection grids when product cards should show market currency.

## Pass `markets` to the cart

Product loaders only change catalog prices. Shopify cart create uses the shop default market unless the same country reaches `ReploProvider` as `markets.shopify.country`. Without it, cards can show EUR while the cart drawer and checkout stay in the shop currency.

```tsx
// app/layout.tsx
import { ReploProvider } from "@replohq/sdk/providers/replo-provider";

export default async function RootLayout({
  children,
}: React.PropsWithChildren) {
  const country = await getMarketCountry();

  return (
    <html lang="en">
      <body>
        <ReploProvider markets={{ shopify: { country } }}>
          {children}
        </ReploProvider>
      </body>
    </html>
  );
}
```

Use the same Storefront `CountryCode` you pass to loaders. When `markets.shopify.country` changes, the existing cart is updated so line prices and checkout follow the new market.

## Behavior notes

- Confirm the merchant has Shopify Markets (or international pricing) enabled for the target country. Otherwise Storefront keeps returning the shop default currency even when `country` / `markets.shopify.country` is set.
- A valid `CountryCode` that is not in a priced market does **not** error on `cartCreate`. Shopify still stores `buyerIdentity.countryCode` as requested and prices the cart in the shop default (or rest-of-world) currency. Do not special-case that as a failure.
- Invalid `CountryCode` values (not in Shopify's enum, including `EU` and `XX`) are rejected by the SDK type and by canopy-api loader/cart validation. Fix the mapping rather than swallowing them.
- URL prefixes like `en-eu` are site routing conventions; they only affect prices after you map them to a concrete `CountryCode` and pass `country` on the loaders.
