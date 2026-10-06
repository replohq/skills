---
name: statsig
title: Statsig Experiments
summary: Run an A/B experiment on a page with Statsig.
description: "REQUIRED before running a Statsig A/B experiment on a page. Contains the mandatory SSR cohorting pattern (stable-ID cookie + data loader) — code generated without reading this skill will flash the wrong variant, mis-bucket visitors, or fail to record results."
---

# Statsig Integration

Statsig is a feature-flag, experimentation, and product-analytics platform. On Replo-published pages you use it to render the variant a visitor is assigned to for an A/B experiment, evaluated server-side so there is no flash of the original content.

## How experiment cohorting works here

The loader evaluates the experiment for a single visitor **server-side** and returns the assigned group and its parameter values. Rendering the variant during SSR (not in the browser) is what avoids a flash of the control variant. Statsig logs the exposure on that evaluation.

Three things make this correct, and you MUST do all of them:

1. **The experiment's own visitor ID.** Statsig credits a conversion to a variant only when the conversion event carries the experiment's ID type and the same value, in the same Statsig project. Send the ID the experiment buckets on, with the value the conversions are logged under.
2. **A durable visitor cookie.** The ID must be the same on every request for that visitor, so they keep the same variant.
3. **Server-side rendering per request.** The page must render per request, not be statically cached; otherwise every visitor gets one cached variant. Reading a cookie with `cookies()` opts the route into dynamic rendering automatically, so you do **not** need `export const dynamic = "force-dynamic"`.

## SDK version prerequisite

`ensureVisitorId` and the loader's `customIds` prop do not exist in every published `@replohq/sdk`. Check that the installed copy exports the helper before writing code:

```sh
node -e "console.log(Object.keys(require('@replohq/sdk/routing/visitor-id')))"
```

If `ensureVisitorId` is missing, run `pnpm add @replohq/sdk@latest` and check again. If it is still missing, the feature has not shipped to npm yet: stop and tell the user.

### Step 1: Find the experiment's ID type

Ask the user for the experiment's **ID Type**, shown in the Statsig console on the experiment's setup page:

- `stableID`: Statsig's built-in anonymous visitor ID. Pass it as `stableId`.
- `userID`: a logged-in user's ID. Pass it as `userId`, read from the customer's own login cookie or session. Never create it with `ensureVisitorId`; render the default for logged-out visitors.
- Anything else, such as `publicID` or `companyID`, is a custom ID type the customer created in their Statsig project. Pass it in `customIds`, keyed by the ID type exactly as Statsig shows it. A different spelling breaks bucketing and attribution without any error.

### Step 2: Find where the visitor ID lives

Ask the user:

- Which cookie holds the visitor ID their own site logs conversions under? For example, a funnel might send its `brand_visitor_id` cookie as `publicID`. When the experiment uses `stableID` and the customer has no visitor cookie of their own, use a `statsig_stable_id` cookie.
- Is that cookie set for the whole domain (`Domain=.brand.com`), or only on the host that sets it? This page can't read a cookie that lives only on another host, and creating a second one splits returning visitors between two IDs. If it is host-only, the customer's site must set it on the parent domain first.
- May this site create that cookie for first-time visitors, or only read it?
- Which domain do conversions happen on, e.g. `www.brand.com` for a page on `lp.brand.com`?

When conversions happen on another host, the page must be served from a custom subdomain of that same domain (for example `lp.brand.com` handing off to `www.brand.com`). A `*.replosites.com` address can never share a cookie with the customer's domain, so the user must connect a subdomain first. Each brand domain needs its own subdomain.

The Statsig client SDK keeps its own stable ID per site and never reads a `statsig_stable_id` cookie. For a `stableID` experiment whose conversions are logged in the browser, the Statsig client must start with `customIDs: { stableID: <statsig_stable_id cookie value> }`, or log conversions through `/v1/log_event` with that value. Do that on this site's pages; when conversions happen on the customer's other site, tell the user their client needs the change.

### Step 3: Create the visitor ID in middleware

`cookies()` inside a page can only **read** cookies, not set them. When the customer allows creating the ID, call `ensureVisitorId` from `@replohq/sdk/routing/visitor-id` on the response the rest of `middleware.ts` returns. Don't hand-write cookie code: the helper never replaces an existing cookie, scopes the new one to `cookieDomain` when the site is served under it, and stops caches from storing the response that sets it.

If `middleware.ts` already exists, change only its final `return`: wrap whatever it returns in `ensureVisitorId`, and keep its redirect rules, `localeRouting` argument, custom redirects, and other logic exactly as they are. If it doesn't exist, create it:

```ts
// middleware.ts
import type { NextRequest } from "next/server";
import type { RedirectRule } from "@replohq/sdk/routing/rules";

import { evaluateRouting } from "@replohq/sdk/routing/evaluate";
import { ensureVisitorId } from "@replohq/sdk/routing/visitor-id";

const rules: RedirectRule[] = [];

export function middleware(request: NextRequest) {
  return ensureVisitorId({
    request,
    response: evaluateRouting({ request, rules }),
    cookieName: "brand_visitor_id",
    cookieDomain: "brand.com",
    requiredConsent: null,
  });
}

// The matcher must remain static so Next.js can detect it at build time.
export const config = {
  matcher: ["/((?!_next/).*)"],
};
```

`cookieDomain` is the registrable domain shared with the conversion host: `brand.com` for `www.brand.com`, `brand.co.uk` for `shop.brand.co.uk`, never a bare suffix such as `co.uk`. Pass `null` when conversions happen on the same host as the page. On any other host, such as a preview, the cookie stays on that host. If one site serves several brand domains, pick the `cookieDomain` that `request.nextUrl.hostname` ends with.

If the customer only allows reading the ID, skip `ensureVisitorId`: first-time visitors see the default and log no exposure.

**Consent.** First check the `scripts` passed to `<ReploScripts>` in `app/layout.tsx`. A `Cookiebot` or `consentPlatform` entry means a third-party tool owns consent and the native mode is `off` on purpose: don't create the ID; leave the site read-only and tell the user why. Otherwise check `data-replo-consent-mode` on `<html>`. When it is `simple` or `per-category`, pass `requiredConsent: "analytics"` instead of `null` here and to `findVisitorId` in Step 4: visitors who haven't accepted analytics see the default and log no exposure, and their next request after accepting gets the ID. Never parse a consent cookie yourself.

### Step 4: Read the ID and prefetch the assignment (Server Component)

```tsx
// app/page.tsx (Server Component)
import { cookies } from "next/headers";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";
import { findVisitorId } from "@replohq/sdk/routing/visitor-id";
import { Hero } from "./Hero";

const EXPERIMENT = "homepage_hero_test";
// The experiment's ID type from Step 1, exactly as Statsig shows it.
const ID_TYPE = "publicID";

export default async function Page() {
  // Reading a cookie makes this route dynamic (per-request); no force-dynamic.
  // Use the same cookieName and requiredConsent as middleware.ts.
  const visitorId = findVisitorId({
    cookies: await cookies(),
    cookieName: "brand_visitor_id",
    requiredConsent: null,
  });
  // Never fall back to a shared ID like "anon": that collapses every
  // cookieless visitor into one variant.
  if (visitorId === null) {
    return <DefaultHero />;
  }
  const customIds = { [ID_TYPE]: visitorId };
  const args = { experimentName: EXPERIMENT, customIds };

  return (
    <PrefetchedLoaders
      queries={[{ loaderKey: DATA_LOADER_KEYS.STATSIG_EXPERIMENT, args }]}
    >
      <Hero customIds={customIds} />
    </PrefetchedLoaders>
  );
}
```

Read the ID with `findVisitorId`, never `cookies().get` directly: it returns `null` until the visitor grants `requiredConsent`, so a visitor who declines or later withdraws consent sees the default and logs no exposure. The page sees the cookie on the very first request too, because cookies set in middleware are visible to `cookies()`. For a `stableID` experiment, pass `stableId: visitorId` instead of `customIds` here and in Step 5.

The most common setup is a `stableID` experiment where the customer has no visitor cookie of their own and conversions are logged on this site. Use `cookieName: "statsig_stable_id"` with `cookieDomain: null` in Step 3, read `statsig_stable_id` here, pass `stableId: visitorId` here and in Step 5 (type `Hero`'s prop as `{ stableId: string }`), and start the Statsig client as Step 2 describes.

### Step 5: Render the assigned variant (Client Component)

Pass the **same** ID props so the prefetched cache hydrates without a refetch (the query key is `[loaderKey, args]`).

```tsx
// app/Hero.tsx (Client Component)
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { StatsigExperimentLoader } from "@replohq/sdk/loaders/statsig-experiment-loader";

const EXPERIMENT = "homepage_hero_test";

export function Hero({ customIds }: { customIds: Record<string, string> }) {
  return (
    <StatsigExperimentLoader
      loaderKey={DATA_LOADER_KEYS.STATSIG_EXPERIMENT}
      experimentName={EXPERIMENT}
      customIds={customIds}
      fallback={<DefaultHero />}
    >
      {(assignment) => {
        // `value` holds the parameters you configured for the assigned group;
        // always pass a default so unconfigured params fall back safely.
        const headline =
          typeof assignment.value.headline === "string"
            ? assignment.value.headline
            : "Welcome";
        return assignment.groupName === "Test" ? (
          <NewHero headline={headline} />
        ) : (
          <DefaultHero headline={headline} />
        );
      }}
    </StatsigExperimentLoader>
  );
}
```

## Assignment shape

`StatsigExperimentLoader`'s render prop receives:

- `groupName` — assigned variant group (e.g. `Control`, `Test`); `null` if unassigned
- `value` — object of parameter values for the assigned group (use with a typed default)
- `ruleId` — the Statsig rule that produced the assignment
- `reason` — evaluation reason (debugging)

Branch on `value` (recommended) or `groupName`. Never hard-code group order — read parameters from `value`.

## Measuring results (exposures + conversions)

- The server-side evaluation logs the **exposure** automatically, which is what makes results appear in Statsig. The result is cached per visitor for 60 seconds, so repeat views in that window don't call Statsig again. Statsig de-dupes exposures either way.
- **Conversions** count when any system in the same Statsig project logs them under the experiment's ID type and value. If the customer's own site already logs its funnel events under that ID, add nothing. Otherwise log conversion events (`add_to_cart`, `purchase`, etc.) client-side with the Statsig client SDK, or the `/v1/log_event` HTTP endpoint, using the same ID.
- Don't add the ID to outbound links. The shared cookie carries it, and many funnels ignore IDs passed in URLs.

## Setup requirements (tell the user if missing)

On-page cohorting needs a **Server Secret key** (preferred) or a **Client SDK key** saved on the Statsig integration, in addition to the Console API key the integration needs to connect. If the loader returns a "not set up for on-page experiments" error, the user must add one in the Integrations settings.

## Constraints

- Do not statically cache experiment pages. Reading the visitor cookie keeps the route dynamic; don't add `generateStaticParams`/`export const revalidate` that would defeat per-visitor rendering.
- Always pass the same ID props to `PrefetchedLoaders` and `StatsigExperimentLoader`.
- Never fall back to a shared placeholder ID (e.g. `"anon"`). If the cookie is missing, render the default/control UI instead of evaluating.
- Provide a `fallback` and read `value` params with defaults so the page renders even if evaluation fails.
