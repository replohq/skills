# Shopify Markets (country / currency)

Shopify **Markets** control which currency, prices, and (when configured) market-specific catalog a Storefront request sees. That is a different concept from **translations**:

| Concept | Loader arg | Storefront `@inContext` | What it changes |
| --- | --- | --- | --- |
| Translation / language | `language` | `language:` | Product titles, descriptions, options, metafield text |
| Market / country | `country` | `country:` | Currency code, money amounts, market-aware catalog |

Pass one, both, or neither — they are independent optional args on `ProductLoader`, `CollectionLoader`, and `CollectionProductsLoader`. Language alone never changes currency; country alone never translates catalog text. For `language`, see [translating-shopify-content.md](translating-shopify-content.md).

## When to use this

- The merchant uses Shopify Markets (or international pricing) and wants prices in a market's currency (e.g. EUR on an `en-eu` URL while the shop default is GBP).
- A page or route represents a market (URL prefix, host, or explicit market switcher), not merely a translated dictionary.
- The user asks for currency, Markets, regional pricing, or "show € / $ / £ for this locale."

Do **not** treat locale routing or dictionary work as Markets work. Site i18n (`reploLocaleRouting`, `dictionaries/*.json`) picks which language copy to show; Markets/`country` picks which money amounts Storefront returns.

## Pass `country` to the loaders

`country` is a Shopify Storefront [`CountryCode`](https://shopify.dev/docs/api/storefront/latest/enums/CountryCode) string (`FR`, `GB`, `US`, …) — ISO 3166-1 alpha-2. There is no `EU` country code.

Map the page's market signal to a real Markets country. For BCP-47 tags with a region (`en-CA`, `fr-FR`), the region subtag usually works. For custom market prefixes without a region (`en-eu`), pick a representative country that is in the merchant's euro (or other) Market — e.g. `FR` or `DE` — after confirming Markets is configured for it:

```tsx
// app/[lang]/products/[handle]/ProductDetail.tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

/** Map a site market signal to a Storefront CountryCode. */
function toShopifyCountry(lang: string): string | undefined {
  const locale = new Intl.Locale(lang);
  if (locale.region) {
    return locale.region.toUpperCase();
  }
  // Custom market prefixes without a region subtag — e.g. en-eu for EUR Markets.
  const marketCountries: Record<string, string> = {
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

## Behavior notes

- Confirm the merchant has Shopify Markets (or international pricing) enabled for the target country. Otherwise Storefront keeps returning the shop default currency even when `country` is set.
- If the requested country is a valid `CountryCode` but not enabled for the shop, Shopify silently serves a supported market context (often the primary). That is expected; do not special-case it.
- Invalid `CountryCode` values (not in Shopify's enum) are real GraphQL errors — fix the mapping rather than swallowing them.
- URL prefixes like `en-eu` are site routing conventions; they only affect prices after you map them to a concrete `CountryCode` and pass `country` on the loaders.
