# Site internationalization

Read this when adding, editing, or debugging translations, additional languages, locale routing, or country/language detection. Only do this work when the user asks for it — never invent locale routing, `/fr` URL schemes, or language switchers unprompted.

If the project uses Shopify, also read [translating-shopify-content.md](../../shopify/references/translating-shopify-content.md) so product/collection loaders receive the correct Storefront `language` (page dictionaries alone do not translate Shopify catalog fields).

Replo Sites express locale routing as one `reploLocaleRouting(...)` config in the selected site's root `middleware.ts`, passed to `evaluateRouting` alongside the redirect rules. Translated copy lives in `dictionaries/<locale>.json` files at the site root. There is no other mechanism: no i18n library, no `proxy.ts`, no per-locale page forks.

## SDK version prerequisite

`reploLocaleRouting` and `@replohq/sdk/routing/geo` do not exist in every published `@replohq/sdk`. A site whose installed copy predates them fails at build with an unresolved export, which reads as a broken middleware rather than a stale dependency — so check before writing any config.

Verify the installed copy actually exports it, rather than trusting the version number:

```sh
node -e "console.log(Object.keys(require('@replohq/sdk/routing/locale')))"
```

If `reploLocaleRouting` is missing, upgrade the site's SDK and re-run the check:

```sh
pnpm add @replohq/sdk@latest
```

Templates depend on `"@replohq/sdk": "latest"`, so a freshly provisioned site is normally current; an older site carries whatever its lockfile pinned. If the export is still missing after upgrading, the feature has not shipped to npm yet — stop and tell the user instead of writing config that cannot resolve, and never hand-write a substitute for `reploLocaleRouting`.

## Middleware config

```ts
import type { NextRequest } from "next/server";

import { evaluateRouting } from "@replohq/sdk/routing/evaluate";
import { reploLocaleRouting } from "@replohq/sdk/routing/locale";
import { reploRedirect } from "@replohq/sdk/routing/rules";

const localeRouting = reploLocaleRouting({
  locales: ["en-CA", "fr-CA"],
  defaultLocale: "en-CA",
  detection: "language", // default; or "country"
  urlMode: "prefixed", // default; or "hidden"
});

const rules = [
  reploRedirect({ source: "/old-page", destination: "/new-page", status: 308 }),
];

export function middleware(request: NextRequest) {
  return evaluateRouting({ request, rules, localeRouting });
}

// The matcher must remain static so Next.js can detect it at build time.
export const config = {
  matcher: ["/((?!_next/).*)"],
};
```

- `locales` — at least two canonical BCP 47 tags (`en-CA`, not `en-ca`; casing is canonicalized, but write the canonical form). Order expresses preference when matching ties.
- `defaultLocale` — must be one of `locales`.
- `detection: "language"` matches the `Accept-Language` header; `"country"` maps the visitor's IP country onto the locales whose region subtag matches (`CA` → `en-CA`, `fr-CA`), breaking multilingual-country ties with `Accept-Language`, and falls through to language matching when the country claims no locale.
- `urlMode: "prefixed"` serves every page at `/{locale}{path}` (a request without a prefix gets a 307 to its negotiated locale); `"hidden"` keeps URLs clean and rewrites internally.
- `domains` (optional) — domain-per-locale scoping: `domains: { "shop.example.ca": { locales: ["en-CA", "fr-CA"] }, "example.fr": { locales: ["fr-FR"] } }`. Keys are bare lowercase hostnames. The Host header picks the scope; a single-locale domain always serves clean URLs with no detection; unknown hosts (previews, sandboxes) use the global scope.

Semantics that are always on — do not reimplement or work around them:

- An explicit choice in the `replo-locale` cookie beats every detection signal. Landing on a locale-prefixed URL sets it.
- Matching is best-fit: a `fr-CA` visitor gets `fr-FR` when that is the only French. A region mismatch never 404s.
- The `replo-country-override` cookie (a 2-letter country code) overrides IP-country detection — use it to test `"country"` mode without a VPN, in dev or production.
- Paths under `/api` and paths whose last segment contains a dot (`favicon.ico`, `sitemap.xml`) are exempt from locale routing.
- Redirect rules match the raw path first, then the locale-stripped path. Write sources against logical paths (`/old-page`) unless you specifically mean one locale — a locale-prefixed source (`/en-CA/something`) targets exactly that locale. Same-site destinations are locale-prefixed in the same response, so redirects land in one hop; a destination you wrote with an explicit locale prefix is left alone.

## Site structure

All pages live under `app/[lang]/` in **both** URL modes — switching `urlMode` is a config edit, never a file move. Dictionaries live at the site root, one file per canonical locale tag, nested string keys, identical key sets across files:

```json
// dictionaries/fr-CA.json
{ "hero": { "title": "Le sac M1", "cta": "Acheter" } }
```

**A dictionary holds nested string maps and nothing else. Never write an array**, even for a list of cards, bullets, or nav items. An array puts the order and the number of items in the copy file, so translating a page could reorder or drop a card — and the Website Builder rejects the whole file, showing a repair prompt instead of the translations table. Numbered keys (`pages.0`) are the same mistake wearing a different hat.

Give each item a name and let the component own the order:

```json
// dictionaries/en-US.json — one key per item, named for what it says
{ "features": { "bfcm": "BFCM campaign pages", "quiz": "Quiz funnels" } }
```

```tsx
// the list lives in the component, so adding an item is a code change
const features = [dict.features.bfcm, dict.features.quiz];
```

Adding, removing, or reordering items is then a code edit, and translating is only ever a wording edit. Writing a dictionary through the `write` or `edit` tool validates it automatically and reports any violation on the tool result; fix it in the same turn.

`app/[lang]/dictionaries.ts` is the loader; keep its map in exact sync with the config's locales:

```ts
import "server-only";

const dictionaries = {
  "en-CA": () => import("../../dictionaries/en-CA.json").then((m) => m.default),
  "fr-CA": () => import("../../dictionaries/fr-CA.json").then((m) => m.default),
};

export type Locale = keyof typeof dictionaries;
export const hasLocale = (locale: string): locale is Locale =>
  locale in dictionaries;
export const getDictionary = async (locale: Locale) => dictionaries[locale]();
```

Pages resolve the dictionary and render from it. The variable holding it MUST be named `dict` — editor tooling recognizes that name when identifying translated text:

```tsx
import { notFound } from "next/navigation";

import { getDictionary, hasLocale } from "../dictionaries";

export default async function Page({
  params,
}: {
  params: Promise<{ lang: string }>;
}) {
  const { lang } = await params;
  if (!hasLocale(lang)) notFound();
  const dict = await getDictionary(lang);
  return <h1>{dict.hero.title}</h1>;
}
```

`app/[lang]/layout.tsx` declares the locales for static generation and the hreflang alternates, using `localeUrl` from `@replohq/sdk/routing/locale` (it returns a prefixed path, or an absolute URL when a `domains` scope serves that locale elsewhere):

```ts
export function generateStaticParams() {
  return [{ lang: "en-CA" }, { lang: "fr-CA" }];
}
```

Language switchers link with `localeUrl({ locale, path, config, currentHost })` too. Never hand-roll switcher logic: in hidden mode the runtime converts the prefixed link into the locale cookie plus a clean-URL redirect on arrival, and with domain scopes it produces the cross-domain URL.

In prefixed mode the generated `app/sitemap.ts` lists every locale's URL with hreflang alternates (and an `x-default` at the unprefixed, locale-negotiating URL). The locale list is embedded at generation time, so after adding or removing languages re-save SEO settings — or regenerate the sitemap — to refresh it. Hidden mode keeps single logical URLs by design: its per-locale URLs are switcher redirects, which never belong in a sitemap.

## Right-to-left languages

Arabic, Hebrew, Persian, Urdu, and others read right-to-left. Adding one is not only a translation: the layout has to mirror, or the page renders with its navigation, arrows, and text alignment on the wrong side. Do this whenever a site gains its first RTL locale.

**Derive the direction from the locale's script, never from a list of language codes.** Script is what actually decides direction, and a language list gets the mixed-script cases wrong: `az-Arab` (Azerbaijani in Arabic script) is RTL while `az-Latn` is LTR, and Sorani Kurdish `ckb` is RTL while Kurmanji `ku` is LTR.

```ts
// lib/direction.ts
const RTL_SCRIPTS = new Set([
  "Arab",
  "Hebr",
  "Thaa",
  "Syrc",
  "Nkoo",
  "Samr",
  "Mand",
  "Adlm",
]);

export function directionForLocale(locale: string): "ltr" | "rtl" {
  try {
    const script = new Intl.Locale(locale).maximize().script ?? "";
    return RTL_SCRIPTS.has(script) ? "rtl" : "ltr";
  } catch {
    return "ltr";
  }
}
```

Use `Intl.Locale#maximize()`, which is stable across Node, workerd, and browsers. Do **not** use `Intl.Locale#textInfo` or `getTextInfo()`: that proposal is still changing shape and the two forms disagree between runtimes, so a site that works in dev can throw in production.

Set `dir` beside `lang` on `<html>` in `app/[lang]/layout.tsx`. That one attribute is what makes every CSS logical property resolve correctly:

```tsx
export default async function LangLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ lang: string }>;
}) {
  const { lang } = await params;
  const dir = directionForLocale(lang);
  return (
    <html lang={lang} dir={dir}>
      {/* DirectionProvider import depends on the site's primitives — see below */}
      <body>
        <DirectionProvider direction={dir}>{children}</DirectionProvider>
      </body>
    </html>
  );
}
```

The `components/ui/` library is shadcn, so shadcn's own RTL docs are the reference for how its components behave — read [the RTL guide](https://ui.shadcn.com/docs/rtl) and [its Next.js setup page](https://ui.shadcn.com/docs/rtl/next) before adapting them. Follow their `<html dir>` and provider structure; ignore the `shadcn/create` and CLI steps for the reason below.

`DirectionProvider` fixes component _behavior_ only (which side a popover opens, which arrow key moves which way); it styles nothing, so the `dir` attribute above is still what makes the layout mirror. **Import it from whichever primitive the site actually uses** — check `package.json` rather than assuming, because both are in the fleet:

Base UI sites (`@base-ui/react` in `package.json`) — most registry site templates:

<!-- prettier-ignore -->
```tsx
import { DirectionProvider } from "@base-ui/react/direction-provider";
<DirectionProvider direction={dir}>{children}</DirectionProvider>
```

Radix sites (`@radix-ui/*` in `package.json`) — the Next.js scaffold most projects start from:

<!-- prettier-ignore -->
```tsx
import { Direction } from "radix-ui";
<Direction.Provider dir={dir}>{children}</Direction.Provider>
```

Note the prop differs: Base UI takes `direction`, Radix takes `dir`. On a Radix site the direction package is usually not installed yet, so add it before importing.

Do **not** run shadcn's `migrate rtl` CLI or set the `rtl` option in `components.json`. These sites vendor their `components/ui/` files and ship no `components.json`, so the CLI has nothing to act on — apply the logical-property edits below to the component files directly instead.

**Check the font covers the script.** Inter and Geist carry no Arabic or Hebrew glyphs, so an RTL locale silently falls back to a system font and the page looks broken next to its LTR version. Load a face that covers the script — Noto Sans Arabic or Noto Sans Hebrew pair well with Inter and Geist — and scope it to the RTL locale rather than swapping the site's font everywhere.

**Write layout classes as logical properties from the start**, so a site is RTL-ready before anyone asks. Tailwind resolves these against `dir`, and they cost nothing in LTR:

| Physical (mirrors wrongly)      | Logical (use this)              |
| ------------------------------- | ------------------------------- |
| `ml-4` / `mr-4`                 | `ms-4` / `me-4`                 |
| `pl-4` / `pr-4`                 | `ps-4` / `pe-4`                 |
| `left-0` / `right-0`            | `start-0` / `end-0`             |
| `text-left` / `text-right`      | `text-start` / `text-end`       |
| `border-l` / `border-r`         | `border-s` / `border-e`         |
| `rounded-l-md` / `rounded-r-md` | `rounded-s-md` / `rounded-e-md` |

Keep physical classes only where the meaning is genuinely physical and must not mirror — a logo's drop shadow, a chart axis, a video control bar. Reach for the `rtl:` and `ltr:` variants for those exceptions rather than abandoning logical properties everywhere.

**Flip directional icons, and only those.** An arrow or chevron that means "forward" points the other way in RTL:

```tsx
<ArrowRight className="rtl:rotate-180" />
```

Do not flip icons whose meaning is not directional — clocks, logos, checkmarks, magnifiers, or a play button — and never mirror the whole page with a CSS transform.

Verify by loading the RTL locale's URL and confirming the page reads right-to-left: navigation starts on the right, text is right-aligned, and forward arrows point left. Check the rendered `<html dir>` matches the locale.

## Editing translated copy

- Add or change copy by editing every `dictionaries/<locale>.json`, translating each value appropriately for its locale — never by forking a page per locale and never by hardcoding one locale's text in JSX.
- Keep key sets identical across locale files. If you cannot produce a translation for a key, copy the default locale's value and report the untranslated keys — never leave a locale file missing keys silently.
- Dictionary file names and `[lang]` values match the config's canonical tags exactly.

## Verification

After editing, reread every changed file, then verify against the running site:

1. An unprefixed path redirects (prefixed mode) or renders (hidden mode) per `Accept-Language`: check `curl -H "Accept-Language: fr-CA" -i` against the default and each configured locale.
2. A `replo-locale` cookie wins over the header; a `replo-country-override` cookie exercises `"country"` mode.
3. A non-canonical prefix casing (`/fr-ca/...`) 308s to the canonical form (prefixed mode).
4. Each locale renders its own dictionary values on a representative page.
5. Exempt paths (`/api/...`, `/sitemap.xml`, `/favicon.ico`) are untouched by locale routing.
6. Existing redirects still fire for a prefixed request (`/fr-CA/old-page` lands on `/fr-CA/new-page` in one hop).
