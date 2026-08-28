# Translating Shopify Content

Page dictionaries (`dictionaries/<locale>.json`) translate Replo site copy only. Shopify product titles, descriptions, options, and metafield values come from the Storefront API and stay in the shop's primary language unless you pass `language` into the product, collection, and collection-products loaders.

## When to use this

- The site already has (or is adding) locale routing via `reploLocaleRouting` — see the localization reference in the **building-replo-pages** skill.
- A page renders Shopify catalog data through `ProductLoader`, `CollectionLoader`, or `CollectionProductsLoader`.
- The merchant has translations in Shopify (Translate & Adapt / Markets). Those values are not separate metafields in admin; Storefront returns them for the same `namespace`/`key` when the request is contextualized.

## Pass `language` to the loaders

`language` is a Shopify Storefront [`LanguageCode`](https://shopify.dev/docs/api/storefront/latest/enums/LanguageCode) string (`FR`, `EN`, `ES`, …), **not** a BCP-47 locale tag like `fr-CA`.

Convert the page's `lang` route param (or the locale you resolved for dictionaries) before passing it:

```tsx
// app/[lang]/products/[handle]/ProductDetail.tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { ProductLoader } from "@replohq/sdk/loaders/product-loader";

function toShopifyLanguage(lang: string): string {
  // fr-CA → FR, en-CA → EN. Most Storefront codes are ISO 639-1 uppercased.
  // Map PT-BR / ZH-CN style tags explicitly when you need PT_BR / ZH_CN.
  return new Intl.Locale(lang).language.toUpperCase();
}

export function ProductDetail({
  lang,
  handle,
}: {
  lang: string;
  handle: string;
}) {
  const language = toShopifyLanguage(lang);

  return (
    <ProductLoader
      loaderKey={DATA_LOADER_KEYS.SHOPIFY_PRODUCT}
      handle={handle}
      language={language}
      fallback={<div>Loading…</div>}
    >
      {(product) => <h1>{product.title}</h1>}
    </ProductLoader>
  );
}
```

Wire the same `language` into `PrefetchedLoaders` on the server page so the React Query key matches:

```tsx
// app/[lang]/products/[handle]/page.tsx
import { notFound } from "next/navigation";

import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { getDictionary, hasLocale } from "../../dictionaries";
import { ProductDetail } from "./ProductDetail";

export default async function Page({
  params,
}: {
  params: Promise<{ lang: string; handle: string }>;
}) {
  const { lang, handle } = await params;
  if (!hasLocale(lang)) notFound();
  const dict = await getDictionary(lang);
  const language = new Intl.Locale(lang).language.toUpperCase();

  return (
    <PrefetchedLoaders
      queries={[
        {
          loaderKey: DATA_LOADER_KEYS.SHOPIFY_PRODUCT,
          args: { handle, language },
        },
      ]}
    >
      <ProductDetail lang={lang} handle={handle} />
      {/* dict still drives non-Shopify copy */}
      <p>{dict.hero.cta}</p>
    </PrefetchedLoaders>
  );
}
```

`CollectionLoader` and `CollectionProductsLoader` accept the same optional `language` prop / args field. Pass it on collection grids too — otherwise product cards keep the shop's primary-language titles/options while the rest of the page is localized.

## Behavior notes

- Country and language are independent. This prop only sets `@inContext(language:)` — it does not change market pricing or country context.
- If the requested language is a valid `LanguageCode` but not enabled for the shop/market, Shopify silently serves a supported language (often the primary). That is expected; do not special-case it.
- Invalid `LanguageCode` values (not in Shopify's enum) are real GraphQL errors — fix the mapping rather than swallowing them.
