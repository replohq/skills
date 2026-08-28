---
name: smile-io-integration
title: Smile.io Loyalty
summary: Build loyalty and rewards pages backed by Smile.io.
description: "REQUIRED before building loyalty landing pages with Smile.io data. Contains mandatory data loading patterns — code generated without reading this skill will use incorrect architecture."
---

# Smile.io Integration

## Data Loader Keys

Always import constants from `@replohq/sdk/loaders/loader-keys`:

- `DATA_LOADER_KEYS.SMILE_IO_VIP_TIERS` — VIP tier definitions
- `DATA_LOADER_KEYS.SMILE_IO_EARNING_RULES` — earning rules (ways to earn points)
- `DATA_LOADER_KEYS.SMILE_IO_REWARDS` — rewards catalog (redeemable items)

These loaders have **no input args** — they load the full program configuration.

## Data Loading Pattern

**Always use the prefetch pattern by default.** The pattern has two layers:

1. **Server component (page):** Uses `PrefetchedLoaders` to fetch data during SSR and seed the React Query cache via `HydrationBoundary`.
2. **Client component:** Uses a loader component (e.g., `SmileVipTiersLoader`) with `useSuspenseQuery` under the hood.

### Example: VIP Tier Benefits Section

**Client component** (`app/components/VipTierBenefits.tsx`):

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { SmileVipTiersLoader } from "@replohq/sdk/loaders/smile-vip-tiers-loader";

export function VipTierBenefits() {
  return (
    <SmileVipTiersLoader
      loaderKey={DATA_LOADER_KEYS.SMILE_IO_VIP_TIERS}
      fallback={<div>Loading tiers…</div>}
    >
      {({ vipTiers }) => (
        <div className="grid grid-cols-3 gap-6">
          {vipTiers.map((tier) => (
            <div
              key={tier.id}
              className="rounded-xl border p-6 text-center"
              style={{ borderColor: tier.color ?? "#e5e7eb" }}
            >
              <h3 className="text-xl font-bold">{tier.name}</h3>
              <p className="mt-2 text-sm text-gray-600">
                {tier.minimumPointsRequired === 0
                  ? "Starting tier"
                  : `${tier.minimumPointsRequired.toLocaleString()} points to unlock`}
              </p>
            </div>
          ))}
        </div>
      )}
    </SmileVipTiersLoader>
  );
}
```

**Server component** (`app/loyalty/page.tsx`):

```tsx
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";
import { VipTierBenefits } from "../components/VipTierBenefits";

export default function LoyaltyPage() {
  return (
    <PrefetchedLoaders
      queries={[
        { loaderKey: DATA_LOADER_KEYS.SMILE_IO_VIP_TIERS, args: {} },
      ]}
    >
      <VipTierBenefits />
    </PrefetchedLoaders>
  );
}
```

### Example: How to Earn Points Section

**Client component** (`app/components/HowToEarnPoints.tsx`):

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { SmileEarningRulesLoader } from "@replohq/sdk/loaders/smile-earning-rules-loader";

export function HowToEarnPoints() {
  return (
    <SmileEarningRulesLoader
      loaderKey={DATA_LOADER_KEYS.SMILE_IO_EARNING_RULES}
      fallback={<div>Loading earning rules…</div>}
    >
      {({ earningRules }) => (
        <div className="space-y-4">
          <h2 className="text-2xl font-bold">How to Earn Points</h2>
          <ul className="space-y-3">
            {earningRules.map((rule) => (
              <li key={rule.id} className="flex items-center justify-between rounded-lg bg-gray-50 p-4">
                <span className="font-medium">{rule.name}</span>
                <span className="text-sm font-semibold text-green-600">
                  {rule.pointsAwarded != null
                    ? `+${rule.pointsAwarded} ${rule.pointsAwardedType === "per_dollar" ? "pts/$" : "pts"}`
                    : "Variable"}
                </span>
              </li>
            ))}
          </ul>
        </div>
      )}
    </SmileEarningRulesLoader>
  );
}
```

### Example: Rewards Catalog Section

**Client component** (`app/components/RewardsCatalog.tsx`):

```tsx
"use client";

import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { SmileRewardsLoader } from "@replohq/sdk/loaders/smile-rewards-loader";

export function RewardsCatalog() {
  return (
    <SmileRewardsLoader
      loaderKey={DATA_LOADER_KEYS.SMILE_IO_REWARDS}
      fallback={<div>Loading rewards…</div>}
    >
      {({ rewards }) => (
        <div className="space-y-4">
          <h2 className="text-2xl font-bold">Rewards</h2>
          <div className="grid grid-cols-2 gap-4 md:grid-cols-3">
            {rewards.map((reward) => (
              <div key={reward.id} className="rounded-xl border p-4 text-center">
                <h3 className="font-semibold">{reward.name}</h3>
                <p className="mt-1 text-lg font-bold text-purple-600">
                  {reward.pointsCost.toLocaleString()} pts
                </p>
                {reward.discountValue && (
                  <p className="mt-1 text-sm text-gray-500">
                    ${reward.discountValue} off
                  </p>
                )}
              </div>
            ))}
          </div>
        </div>
      )}
    </SmileRewardsLoader>
  );
}
```

### Full Loyalty Landing Page with All Loaders

Combine multiple loaders on a single page by listing them all in `PrefetchedLoaders`:

```tsx
import { DATA_LOADER_KEYS } from "@replohq/sdk/loaders/loader-keys";
import { PrefetchedLoaders } from "@replohq/sdk/loaders/prefetch-loaders";
import { VipTierBenefits } from "../components/VipTierBenefits";
import { HowToEarnPoints } from "../components/HowToEarnPoints";
import { RewardsCatalog } from "../components/RewardsCatalog";

export default function LoyaltyPage() {
  return (
    <PrefetchedLoaders
      queries={[
        { loaderKey: DATA_LOADER_KEYS.SMILE_IO_VIP_TIERS, args: {} },
        { loaderKey: DATA_LOADER_KEYS.SMILE_IO_EARNING_RULES, args: {} },
        { loaderKey: DATA_LOADER_KEYS.SMILE_IO_REWARDS, args: {} },
      ]}
    >
      <section className="space-y-16 py-12">
        <VipTierBenefits />
        <HowToEarnPoints />
        <RewardsCatalog />
      </section>
    </PrefetchedLoaders>
  );
}
```

## Data Shapes

### SmileVipTier

| Field                  | Type             | Description                    |
| ---------------------- | ---------------- | ------------------------------ |
| id                     | number           | Tier ID                        |
| name                   | string           | Tier name (e.g. "Gold")        |
| minimumPointsRequired  | number           | Points threshold to reach tier |
| color                  | string \| null   | Hex color code                 |

### SmileEarningRule

| Field              | Type   | Description                              |
| ------------------ | ------ | ---------------------------------------- |
| id                 | number | Rule ID                                  |
| name               | string | Display name (e.g. "Make a purchase")    |
| pointsAwarded      | number \| null | Points awarded per action (null for variable rules) |
| pointsAwardedType  | string | "fixed" or "per_dollar"                  |
| activityToken      | string | Internal activity identifier             |

### SmileReward

| Field         | Type           | Description                          |
| ------------- | -------------- | ------------------------------------ |
| id            | number         | Reward ID                            |
| name          | string         | Display name (e.g. "$5 Off Coupon")  |
| pointsCost    | number         | Points required to redeem            |
| rewardType    | string         | Reward type identifier               |
| discountValue | string \| null | Discount amount (if applicable)      |

## Empty Data Handling

When a loader returns an empty array, the loyalty program likely doesn't have that feature configured:

- `{ vipTiers: [] }` — VIP tiers not set up
- `{ earningRules: [] }` — No earning rules enabled
- `{ rewards: [] }` — No rewards configured

Render a helpful message or hide the section when empty.
