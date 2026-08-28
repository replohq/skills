---
name: statsig
title: Statsig Experiments
summary: Run an A/B experiment on a page with Statsig.
description: "REQUIRED before running a Statsig A/B experiment on a page. Contains the mandatory SSR cohorting pattern (stable-ID cookie + data loader) — code generated without reading this skill will flash the wrong variant, mis-bucket visitors, or fail to record results."
---

# Statsig Integration

Statsig is a feature-flag, experimentation, and product-analytics platform. On Replo-published pages you use it to render the variant a visitor is assigned to for an A/B experiment, evaluated server-side so there is no flash of the original content.

## How experiment cohorting works here

The loader evaluates the experiment for a single visitor **server-side** and returns the assigned group and its parameter values. Rendering the variant during SSR (not in the browser) is what avoids a flash of the control variant.

Two things make this correct, and you MUST do both:

1. **A stable visitor ID.** Bucketing is deterministic on a per-visitor ID. It must be the same value on every request for that visitor (so they keep the same variant) and match the ID used for any client-side event logging (so conversions attribute to the right group). Use a `statsig_stable_id` cookie.
2. **Server-side rendering per request.** The page must render per request, not be statically cached — otherwise every visitor gets one cached variant. Reading a cookie with `cookies()` opts the route into dynamic rendering automatically, so you do **not** need `export const dynamic = "force-dynamic"`. Just call `cookies()`.

### Step 1: Mint the stable ID cookie in middleware

`cookies()` inside a page can only **read** cookies, not set them. Mint the cookie once in `middleware.ts` so first-time visitors get a stable ID before the page renders:

```ts
// middleware.ts
import { NextResponse, type NextRequest } from "next/server";

const STABLE_ID_COOKIE = "statsig_stable_id";

export function middleware(request: NextRequest) {
  const response = NextResponse.next();
  if (!request.cookies.get(STABLE_ID_COOKIE)) {
    response.cookies.set(STABLE_ID_COOKIE, crypto.randomUUID(), {
      httpOnly: false, // readable by the Statsig client SDK for event logging
      sameSite: "lax",
      maxAge: 60 * 60 * 24 * 365,
      path: "/",
    });
  }
  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### Step 2: Read the cookie and prefetch the assignment (Server Component)

```tsx
// app/page.tsx (Server Component)
import { cookies } from "next/headers";
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";
import { Hero } from "./Hero";

const EXPERIMENT = "homepage_hero_test";

export default async function Page() {
  // Reading a cookie makes this route dynamic (per-request) — no force-dynamic.
  const stableId = (await cookies()).get("statsig_stable_id")?.value;
  // Middleware should have minted the cookie. Never fall back to a shared ID
  // like "anon" — that collapses every cookieless visitor into one variant.
  if (stableId == null) {
    return <DefaultHero />;
  }
  const args = { experimentName: EXPERIMENT, stableId };

  return (
    <PrefetchedLoaders
      queries={[{ loaderKey: DATA_LOADER_KEYS.STATSIG_EXPERIMENT, args }]}
    >
      <Hero stableId={stableId} />
    </PrefetchedLoaders>
  );
}
```

### Step 3: Render the assigned variant (Client Component)

Pass the **same** `stableId` so the prefetched cache hydrates without a refetch (the query key is `[loaderKey, args]`).

```tsx
// app/Hero.tsx (Client Component)
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { StatsigExperimentLoader } from "@replohq/sdk/loaders/statsig-experiment-loader";

const EXPERIMENT = "homepage_hero_test";

export function Hero({ stableId }: { stableId: string }) {
  return (
    <StatsigExperimentLoader
      loaderKey={DATA_LOADER_KEYS.STATSIG_EXPERIMENT}
      experimentName={EXPERIMENT}
      stableId={stableId}
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

- The server-side evaluation logs the **exposure** automatically, which is what makes results appear in Statsig. The loader does not use the shared in-memory loader cache (per-visitor keys would unbounded-grow it); every request re-evaluates. Statsig de-dupes exposures, so repeat evaluations for the same visitor are fine.
- **Conversions must still be logged** for results to mean anything. Log conversion events (`add_to_cart`, `purchase`, etc.) client-side with the Statsig client SDK — or the `/v1/log_event` HTTP endpoint — using the **same** `statsig_stable_id` value, or Statsig can't attribute them to the assigned group.

## Setup requirements (tell the user if missing)

On-page cohorting needs a **Server Secret key** (preferred) or a **Client SDK key** saved on the Statsig integration, in addition to the Console API key used by the chat tools. If the loader returns a "not set up for on-page experiments" error, the user must add one in the Integrations settings.

## Constraints

- Do not statically cache experiment pages. Reading the `statsig_stable_id` cookie keeps the route dynamic; don't add `generateStaticParams`/`export const revalidate` that would defeat per-visitor rendering.
- Always pass the same `stableId` to `PrefetchedLoaders` and `StatsigExperimentLoader`.
- Never fall back to a shared placeholder ID (e.g. `"anon"`). If the cookie is missing, render the default/control UI instead of evaluating.
- Provide a `fallback` and read `value` params with defaults so the page renders even if evaluation fails.
