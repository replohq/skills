---
name: querying-analytics
title: Query Analytics
summary: Query a project's traffic, conversion, and revenue data with SQL.
description: |
  Guide for using the `query_replo_analytics` MCP tool to run ClickHouse SQL queries against Replo analytics data.
  Reference when: the agent needs to query analytics data, understand page views, sessions, purchases, conversion rates, or build custom analytics queries.
  Also load for pixel-related queries, e.g. when querying for specific meta ads/adsets, etc
tools: query_replo_analytics
---

# MCP Analytics Query Tool

The `query_replo_analytics` tool lets you run read-only ClickHouse SQL queries against this project's Replo analytics data. Pass a `query` string containing a `SELECT` statement (or a `WITH` CTE that resolves to a `SELECT`).

## Project Scoping (Important)

Queries are **automatically scoped to the current project and any sibling projects in the same workspace that share its Shopify store** by the server. Every query is executed with a per-project ClickHouse role, and the namespaced tables (`events_computed`, `namespace_to_domain`, `daily_page_rollups`, `daily_namespace_rollups`, `daily_namespace_purchase_rollups`) all have RESTRICTIVE row policies that filter rows to those namespaces before they reach you. **You do not need to add `WHERE namespace = ...` filters to your queries.**

Because Shopify's web pixel stamps every conversion event with the single project that first activated the pixel on a store, a store's analytics can live under a sibling project's namespace. Scoping to every workspace sibling that connects the same Shopify store makes that shared-store data visible from any of them. Practically this means: rows can carry **more than one `namespace`**, and `namespace_to_domain` can map to **multiple domains** — so filter/group by `domain` (or `namespace`) when you need to isolate a single storefront.

## Tables

The agent has SELECT access to exactly seven tables (plus optional external-integration metric view functions — see below). Anything else will fail with a permission error:

| Table                              | Description                                                                                                                                                                                              | Row-policied?                |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `events_computed`                  | Session-level computed analytics (PRIMARY TABLE FOR AD-HOC QUERIES)                                                                                                                                      | Yes — scoped to your project |
| `namespace_to_domain`              | Maps namespace → domain. Useful for resolving domain names.                                                                                                                                              | Yes — scoped to your project |
| `currency_exchange_rates`          | Currency exchange rates for multi-currency support.                                                                                                                                                      | No (global)                  |
| `daily_page_rollups`               | Pre-aggregated daily metrics per (day, domain, path, utm_source). USD-converted. **Prefer this for date-range page analytics over `events_computed`.**                                                   | Yes — scoped to your project |
| `daily_namespace_rollups`          | Pre-aggregated daily traffic metrics per (day, utm_source). Views, sessions, converting sessions.                                                                                                        | Yes — scoped to your project |
| `daily_namespace_purchase_rollups` | Pre-aggregated daily purchase metrics per (day, utm_source). Purchase count, USD revenue, unique purchasing sessions.                                                                                    | Yes — scoped to your project |
| `consent_events`                   | Proof-of-consent audit log: one append-only row per visitor consent decision (`action`, `consent_mode`, `policy_version`, `categories` JSON, `decided` via `date`). Use for GDPR consent exports/audits. | Yes — scoped to your project |

### When to use rollups vs `events_computed`

The three `daily_*_rollups` tables are **`AggregatingMergeTree`** tables refreshed every 5 minutes by background materialized views, with a **3-hour data lag** (the most recent 3 hours of data is not yet in the rollups). They are dramatically faster than `events_computed` for any query that fits their grain (daily, by namespace + optional domain/path/utm_source) and is looking for the data that they roll-up over.

**Prefer rollups when:**

- The question is "how many X per day for the last N days/weeks/months?" where X is represented in the rollup
- You need cross-day aggregates (revenue, sessions, conversions) over a non-trivial time range.
- You're grouping by `path`, `domain`, or `utm_source`.

**Fall back to `events_computed FINAL` when:**

- You need data from the last ~3 hours (rollups lag).
- You need a dimension the rollups don't carry (referrer, individual line items, custom event fields, purchase data, hourly granularity).
- You need session-level reasoning (a particular `subject`'s journey).

### `daily_page_rollups` — Per-page daily aggregates

`AggregatingMergeTree` keyed on `(day, namespace, domain, path, utm_source)`. All revenue is **already converted to USD** by the materialized view via `convert_to_usd`.

Aggregate-state columns (must be read with the matching `*Merge` function — see "Reading aggregate-state columns" below):

| Column                         | Combinator   | Meaning                                                                                                             |
| ------------------------------ | ------------ | ------------------------------------------------------------------------------------------------------------------- |
| `views_state`                  | `countMerge` | Total page views                                                                                                    |
| `unique_sessions_state`        | `uniqMerge`  | Unique sessions that viewed this page                                                                               |
| `converting_sessions_state`    | `uniqMerge`  | Unique sessions that viewed this page and ended up purchasing                                                       |
| `fractional_conversions_state` | `sumMerge`   | Page's fractional share of conversions across the funnel (sums across pages = real total)                           |
| `fractional_revenue_usd_state` | `sumMerge`   | Page's fractional share of revenue in USD (sums across pages = real total)                                          |
| `session_conversions_state`    | `sumMerge`   | Total conversions from sessions that viewed this page (overcounts when summed across pages — measures influence)    |
| `session_revenue_usd_state`    | `sumMerge`   | Total revenue from sessions that viewed this page in USD (overcounts when summed across pages — measures influence) |

**Use `fractional_revenue_usd_state` for "total revenue split across the funnel"** — it sums to the real total. Use `session_revenue_usd_state` for "what's the influence of this page" — it overcounts because a multi-page session contributes its full revenue to every page it touched.

### `daily_namespace_rollups` — Per-day traffic aggregates

`AggregatingMergeTree` keyed on `(day, namespace, utm_source)`. Traffic-only (no revenue / conversions); join with `daily_namespace_purchase_rollups` for purchase metrics.

| Column                      | Combinator   | Meaning                                  |
| --------------------------- | ------------ | ---------------------------------------- |
| `views_state`               | `countMerge` | Total page views                         |
| `unique_sessions_state`     | `uniqMerge`  | Unique sessions                          |
| `converting_sessions_state` | `uniqMerge`  | Unique sessions that ended up purchasing |

### `daily_namespace_purchase_rollups` — Per-day purchase aggregates

`AggregatingMergeTree` keyed on `(day, namespace, utm_source)`. Sourced directly from `replo.purchase` events (separate from `daily_namespace_rollups` because purchase metrics need different attribution logic).

| Column                             | Combinator   | Meaning                                  |
| ---------------------------------- | ------------ | ---------------------------------------- |
| `purchase_count_state`             | `countMerge` | Number of purchase events                |
| `total_revenue_usd_state`          | `sumMerge`   | Total revenue in USD (already converted) |
| `unique_purchasing_sessions_state` | `uniqMerge`  | Unique sessions that made a purchase     |

### Reading aggregate-state columns

`AggregatingMergeTree` stores intermediate aggregation state (e.g. for `uniq`, the HLL sketch; for `sum`, a partial sum). To get a usable value, you **must** apply the matching `-Merge` combinator and `GROUP BY` your dimensions. You cannot select a `*_state` column directly and get a number.

Wrong:

```sql
-- This returns binary state blobs, not numbers.
SELECT views_state FROM daily_page_rollups WHERE day = today() - 1
```

Right:

```sql
SELECT
  day,
  path,
  countMerge(views_state) AS views,
  uniqMerge(unique_sessions_state) AS sessions,
  sumMerge(fractional_revenue_usd_state) AS revenue_usd
FROM daily_page_rollups
WHERE day >= today() - 7
GROUP BY day, path
ORDER BY revenue_usd DESC
LIMIT 20
```

The `-If` variant of the underlying aggregate (e.g. `uniqStateIf`) is also stored; you still merge with the plain `-Merge` (e.g. `uniqMerge`), not `uniqMergeIf`.

### `events_computed` — Session-level computed analytics

This is a `ReplacingMergeTree` table keyed on `(namespace, subject, date, id)` with `computedVersion` as the version column. The flusher computes session-level aggregates (purchases, attribution) and writes them into the `computed` JSON column.

**Querying `events_computed`: avoid bare `FINAL`, and never wrap `date` in `toDate(...)`.**

`events_computed` lives on ClickHouse Cloud (object storage). Two query shapes will silently make a "fast empty" query take 10+ seconds:

1. **`FROM events_computed FINAL` with no merge-pruning settings.** `FINAL` performs the ReplacingMergeTree merge at query time. The planner has to open every active part in the partition range (a high-latency S3 round trip per part) before any `WHERE` is applied. Empty result sets pay the same cost as huge ones.
2. **`WHERE toDate(date) BETWEEN ... AND ...`.** Wrapping `date` in `toDate(...)` defeats partition pruning — the planner often cannot recognize the transform as monotonic over the partition key.

Prefer this pattern instead:

```sql
SELECT ...
FROM events_computed FINAL
WHERE date >= toDate('2026-05-25')
  AND date < toDate('2026-05-26') + INTERVAL 1 DAY
  AND ...
SETTINGS do_not_merge_across_partitions_select_final = 1
```

- The half-open range on bare `date` preserves partition pruning. (`date` is `DateTime64(3)`; ClickHouse converts the `Date` literal to a `DateTime` at start-of-day for the comparison.)
- `do_not_merge_across_partitions_select_final = 1` lets ClickHouse skip the cross-partition merge — usually near-free on Cloud when each partition is already mostly merged.

**When you can drop `FINAL` entirely:** if your aggregates are all `uniq*` (sessions, unique visitors), duplicates from un-merged rows don't change the result, so `FINAL` is unnecessary. For `count()` / `sum()` aggregates, duplicates will inflate results — keep `FINAL` (with the settings flag above) or do an explicit dedup subquery:

```sql
SELECT ...
FROM (
  SELECT id, argMax(name, computedVersion) AS name, argMax(subject, computedVersion) AS subject, ...
  FROM events_computed
  WHERE date >= toDate('2026-05-25') AND date < toDate('2026-05-26') + INTERVAL 1 DAY
  GROUP BY id
)
```

| Column            | Type          | Description                                               |
| ----------------- | ------------- | --------------------------------------------------------- |
| `namespace`       | String        | Project namespace (auto-filtered by row policy)           |
| `subject`         | UUID          | Session ID — groups events from the same browsing session |
| `date`            | DateTime64(3) | Event timestamp                                           |
| `id`              | UUID          | Unique event ID                                           |
| `name`            | String        | Event type (see below)                                    |
| `data`            | String        | JSON payload with event-specific fields                   |
| `computed`        | String        | JSON with session-level aggregates (see below)            |
| `computedVersion` | UInt8         | Version for ReplacingMergeTree dedup                      |
| `computedDate`    | DateTime64(3) | When computed values were last updated                    |
| `insertDate`      | DateTime64(3) | Row insertion time                                        |

**Event types** (`name` column):

- `replo.page_view` — Page view
- `replo.session_start` — New session started
- `replo.first_visit` — First-ever visit
- `replo.click` — Generic click
- `replo.custom_click` — Custom click event
- `replo.view_product` — Product detail page view
- `replo.view_collection` — Collection page view
- `replo.add_to_cart` — Added item to cart
- `replo.remove_from_cart` — Removed item from cart
- `replo.view_cart` — Viewed cart
- `replo.start_checkout` — Started checkout
- `replo.purchase` — Completed purchase

**Common `data` JSON fields** (access via `JSONExtract*`):

- `data.root.path` — URL path (e.g. `/products/widget`)
- `data.root.domain` — Domain (e.g. `example.com`)
- `data.root.url` — Full URL
- `data.root.title` — Page title
- `data.root.referrer` — Referrer URL
- `data.root.useragent` — User agent string
- `data.root.params` — URL query parameters object (access individual params with `JSONExtractString(data, 'root', 'params', 'utm_source')`)

**Add-to-cart / commerce payload fields** (on `replo.add_to_cart`, under `data.payload` — not `data.root`):

- `data.payload.productId` / `variantId` / `productTitle` / `variantTitle`
- `data.payload.price` / `currency` / `quantity`

**Purchase event `data.payload` fields:**

- `data.payload.totalPrice.amount` — Total order amount (Float64)
- `data.payload.totalPrice.currencyCode` — Currency code (e.g. `USD`)
- `data.payload.subtotalPrice.amount` — Subtotal before shipping/tax
- `data.payload.lineItems` — Array of purchased items
- `data.payload.lineItems[].title` — Product title
- `data.payload.lineItems[].quantity` — Quantity purchased
- `data.payload.lineItems[].variant.price.amount` — Unit price
- `data.payload.lineItems[].variant.product.title` — Product name
- `data.payload.lineItems[].finalLinePrice.amount` — Line total after discounts
- `data.payload.discountsAmount.amount` — Total discount amount
- `data.payload.order.id` — Shopify order ID
- `data.payload.order.customer.isFirstOrder` — Whether this was the customer's first order

**Materialized columns** (extracted from `data` and `computed` for efficient filtering):

| Column                         | Type    | Source                                    | Description                                                  |
| ------------------------------ | ------- | ----------------------------------------- | ------------------------------------------------------------ |
| `path`                         | String  | `data.root.path`                          | URL path                                                     |
| `domain`                       | String  | `data.root.domain`                        | Domain                                                       |
| `title`                        | String  | `data.root.title`                         | Page title                                                   |
| `currencyCode`                 | String  | `computed.subject.purchase.currencyCode`  | Purchase currency                                            |
| `subjectPurchaseSum`           | Float64 | `computed.subject.purchase.sum`           | Total purchase amount for this **session**                   |
| `subjectPurchaseCount`         | Float64 | `computed.subject.purchase.count`         | Number of purchases in this **session**                      |
| `subjectPurchaseFraction`      | Float64 | `computed.subject.purchase.fraction`      | Fractional attribution weight for this **session**           |
| `subjectPurchaseFractionalSum` | Float64 | `computed.subject.purchase.fractionalSum` | Fractionally attributed purchase amount for this **session** |

**CRITICAL: The `subject*` columns are per-SESSION aggregates, not per-event.**
A `subject` (= session UUID) groups multiple events from the same browsing session. The `subjectPurchaseSum` is the total purchase amount for the _entire session_, repeated on every event row in that session. If you naively `sum(subjectPurchaseSum)`, you will massively overcount because every page view, click, and other event in a purchasing session carries the same session-level total.

To get correct totals, always deduplicate by session first:

```sql
-- CORRECT: deduplicate by session
SELECT sum(purchase_total) AS revenue
FROM (
  SELECT subject, any(subjectPurchaseSum) AS purchase_total
  FROM events_computed FINAL
  WHERE subjectPurchaseCount > 0
    AND date >= '2026-03-19'
  GROUP BY subject
)

-- WRONG: this overcounts because each event row repeats the session total
SELECT sum(subjectPurchaseSum) AS revenue
FROM events_computed FINAL
WHERE subjectPurchaseCount > 0
```

Alternatively, use `uniq(subject)` to count purchasing sessions and `sumIf` with a single event type to avoid double-counting:

```sql
SELECT
  uniqIf(subject, subjectPurchaseCount > 0) AS purchasing_sessions,
  -- Use session_start (one per session) to avoid overcounting
  sumIf(subjectPurchaseSum, name = 'replo.session_start' AND subjectPurchaseCount > 0) AS revenue
FROM events_computed FINAL
WHERE date >= '2026-03-19'
```

**`computed` JSON structure:**

```json
{
  "subject": {
    "purchase": {
      "sum": 81.4,
      "count": 1,
      "fraction": 0.5,
      "fractionalSum": 40.7,
      "currencyCode": "USD"
    }
  }
}
```

The materialized columns are the preferred way to access computed values — they're indexed and faster than `JSONExtract` on the `computed` column.

## External integration metrics

Some connected integrations expose parameterized **view functions** you can `SELECT` from in this same ClickHouse surface (in addition to the Replo pixel tables above). They are live proxies of that integration's metrics — credentials are injected server-side, so never pass auth tokens or project ids. Required args are always `since` / `until` (`YYYY-MM-DD`, inclusive). Integration not connected (or no data) → **zero rows**, not an error.

Today that includes **Triple Whale** (`triplewhale_metrics_daily` / `_totals` / `_list`) and **Contentsquare** (`contentsquare_metrics_daily` / `_totals` / `_list`). Query these views the same way as any other: inspect a view's columns before writing against it, since each integration names its metrics differently.

## Third-party Pixel Comparisons

The user may ask about querying third-party session data, such as Meta ads/adsets, etc. In these cases, it's important to be strategic about queries, especially when comparing with Replo analytics data.

### Meta

If the user asks about meta ad/adset sessions, or why Meta Pixel numbers disagree with Replo, **read [references/meta.md](references/meta.md) before answering**. There are several query patterns or misunderstandings of how pixel setups work that could cause discrepancies.

## Common Query Patterns

### Rollup-based queries (preferred for large date-range aggregates)

#### Daily site traffic over the last 30 days

```sql
SELECT
  day,
  countMerge(views_state) AS views,
  uniqMerge(unique_sessions_state) AS sessions,
  uniqMerge(converting_sessions_state) AS purchasing_sessions
FROM daily_namespace_rollups
WHERE day >= today() - 30
GROUP BY day
ORDER BY day
```

#### Daily revenue and purchases over the last 30 days

```sql
SELECT
  day,
  countMerge(purchase_count_state) AS purchases,
  sumMerge(total_revenue_usd_state) AS revenue_usd,
  uniqMerge(unique_purchasing_sessions_state) AS purchasing_sessions
FROM daily_namespace_purchase_rollups
WHERE day >= today() - 30
GROUP BY day
ORDER BY day
```

#### Top revenue-driving pages over the last 30 days

```sql
SELECT
  path,
  sumMerge(fractional_revenue_usd_state) AS attributed_revenue_usd,
  countMerge(views_state) AS views,
  uniqMerge(unique_sessions_state) AS sessions,
  uniqMerge(converting_sessions_state) AS purchasing_sessions,
  purchasing_sessions / nullIf(sessions, 0) AS conversion_rate
FROM daily_page_rollups
WHERE day >= today() - 30
GROUP BY path
ORDER BY attributed_revenue_usd DESC
LIMIT 20
```

`fractional_revenue_usd_state` is the right column for "real total revenue split across pages" — it sums to the actual revenue. Use `session_revenue_usd_state` if you specifically want "what's the total revenue of every session that touched this page" (will overcount when summed).

#### Traffic + revenue by UTM source

`daily_namespace_rollups` and `daily_namespace_purchase_rollups` share `(day, namespace, utm_source)` so you can join them at that grain:

```sql
SELECT
  t.utm_source AS utm_source,
  uniqMerge(t.unique_sessions_state) AS sessions,
  sumMerge(p.total_revenue_usd_state) AS revenue_usd,
  countMerge(p.purchase_count_state) AS purchases
FROM daily_namespace_rollups t
LEFT JOIN daily_namespace_purchase_rollups p
  ON t.day = p.day AND t.utm_source = p.utm_source
WHERE t.day >= today() - 30
GROUP BY utm_source
ORDER BY revenue_usd DESC
LIMIT 20
```

### Filtering out non-content paths

Checkout, cart, and account pages are typically not useful for content analytics. Filter them out:

```sql
AND path NOT LIKE '/checkout%'
AND path NOT LIKE '/checkouts%'
AND path NOT LIKE '/cart%'
AND path NOT LIKE '/account%'
AND path NOT LIKE '/login%'
AND path NOT LIKE '/password%'
AND path != '/challenge'
```

### Grouping by page

Use the materialized `path` column on `events_computed`:

```sql
SELECT
  path,
  count() AS page_views,
  uniq(subject) AS unique_sessions
FROM events_computed FINAL
WHERE name = 'replo.page_view'
  AND date >= '2026-03-19'
GROUP BY path
ORDER BY page_views DESC
LIMIT 20
```

### Grouping by day

```sql
SELECT toDate(date) AS day, ...
GROUP BY day
ORDER BY day
```

### Filling gaps in time-series queries (`WITH FILL`)

When grouping by day / hour / week, days where nothing happened **drop out of the result entirely** (they're not in the source data, so `GROUP BY` skips them). For dashboards, charts, week-over-week comparisons, and anything that needs a continuous time axis, use ClickHouse's [`WITH FILL`](https://clickhouse.com/docs/guides/developer/time-series-filling-gaps) clause to materialize the missing buckets as zero-valued rows.

The pattern is `ORDER BY <time_bucket> ASC WITH FILL FROM <start> TO <end> STEP <interval>` (the `FROM` and `TO` are optional — without them you only fill gaps inside the existing range, not at the edges):

```sql
SELECT
  day,
  countMerge(views_state) AS views,
  uniqMerge(unique_sessions_state) AS sessions
FROM daily_namespace_rollups
WHERE day >= today() - 30
GROUP BY day
ORDER BY day ASC
WITH FILL
  FROM today() - 30
  TO today() + 1
  STEP INTERVAL 1 DAY
```

Without `WITH FILL`, a 30-day query that has data on only 18 days returns 18 rows. With `WITH FILL`, you get all 31 rows (including the start and end-of-range days), with `views = 0` / `sessions = 0` on the days where nothing happened.

**`STEP INTERVAL` choices:**

- Daily: `STEP INTERVAL 1 DAY` (use with `toDate(...)` or rollup `day`)
- Hourly: `STEP INTERVAL 1 HOUR` (use with `toStartOfHour(date)`)
- Weekly: `STEP INTERVAL 1 WEEK` (use with `toStartOfWeek(date)`)
- Monthly: `STEP INTERVAL 1 MONTH` (use with `toStartOfMonth(date)`)

**`TO` is exclusive** — to include today as a bucket, write `TO today() + 1` rather than `TO today()`.

**Do NOT cast the time-bucket column to a string in the SELECT.** `WITH FILL FROM <start> TO <end>` compares the `ORDER BY` column against the `FROM` / `TO` bounds, and a `String`-aliased column has no supertype with the `Date` / `DateTime` bounds. Wrapping the column in `toString(...)` (or `formatDateTime(...)`, etc.) makes ClickHouse fail with `There is no supertype for types String, Date ...`. Keep the column native (`day`, `toDate(date)`, `toStartOfHour(date)`) — ClickHouse's JSON output serialises Date / DateTime as ISO strings on the wire anyway, so the application layer still receives a string. Format for display in your client code, not in SQL:

```sql
-- WRONG: causes "There is no supertype for types String, Date" because the
-- aliased `day` column is now String but `today() - 6` and `today() + 1` are Date.
SELECT toString(day) AS day, ... FROM daily_namespace_rollups
WHERE day >= today() - 6
GROUP BY day
ORDER BY day ASC
WITH FILL FROM today() - 6 TO today() + 1 STEP INTERVAL 1 DAY

-- RIGHT: keep the column as Date; ClickHouse serialises it to "YYYY-MM-DD"
-- in JSON output regardless.
SELECT day, ... FROM daily_namespace_rollups
WHERE day >= today() - 6
GROUP BY day
ORDER BY day ASC
WITH FILL FROM today() - 6 TO today() + 1 STEP INTERVAL 1 DAY
```

**For running totals over filled-in days**, add `INTERPOLATE` so the cumulative column carries the previous value across the zero-row days instead of resetting to 0:

```sql
SELECT
  day,
  countMerge(purchase_count_state) AS purchases,
  sumMerge(total_revenue_usd_state) AS revenue_usd,
  sum(revenue_usd) OVER (ORDER BY day) AS cumulative_revenue_usd
FROM daily_namespace_purchase_rollups
WHERE day >= today() - 30
GROUP BY day
ORDER BY day ASC
WITH FILL
  FROM today() - 30
  TO today() + 1
  STEP INTERVAL 1 DAY
INTERPOLATE (cumulative_revenue_usd)
```

### Computing purchase metrics

Use the materialized columns on `events_computed`:

```sql
SELECT
  toDate(date) AS day,
  uniqIf(subject, subjectPurchaseCount > 0) AS purchasing_sessions,
  sumIf(subjectPurchaseSum, subjectPurchaseCount > 0) AS revenue
FROM events_computed FINAL
WHERE name = 'replo.page_view'
  AND date >= '2026-03-19'
GROUP BY day
ORDER BY day
```

### Average order value

```sql
SELECT
  sumIf(subjectPurchaseSum, subjectPurchaseCount > 0) /
    nullIf(uniqIf(subject, subjectPurchaseCount > 0), 0) AS avg_order_value
FROM events_computed FINAL
WHERE name = 'replo.page_view'
  AND date >= '2026-03-19'
  AND date <= '2026-04-02'
```

### Conversion rate by page

```sql
SELECT
  path,
  uniq(subject) AS sessions,
  uniqIf(subject, subjectPurchaseCount > 0) AS purchasing_sessions,
  purchasing_sessions / nullIf(sessions, 0) AS conversion_rate,
  sumIf(subjectPurchaseSum, subjectPurchaseCount > 0) AS revenue
FROM events_computed FINAL
WHERE name = 'replo.page_view'
  AND date >= '2026-03-19'
  AND path NOT LIKE '/checkout%'
  AND path NOT LIKE '/cart%'
  AND path NOT LIKE '/account%'
GROUP BY path
ORDER BY revenue DESC
LIMIT 20
```

### UTM source attribution

```sql
SELECT
  JSONExtractString(data, 'root', 'params', 'utm_source') AS utm_source,
  uniq(subject) AS sessions,
  uniqIf(subject, subjectPurchaseCount > 0) AS purchasers,
  sumIf(subjectPurchaseSum, subjectPurchaseCount > 0) AS revenue
FROM events_computed FINAL
WHERE name = 'replo.page_view'
  AND date >= '2026-03-19'
  AND utm_source != ''
GROUP BY utm_source
ORDER BY revenue DESC
LIMIT 10
```

When matching a **Meta ad set / campaign name** from a screenshot to `utm_term` /
`utm_campaign`, normalize percent-encoding (`%2B`→`+`, `%3D`→`=`) — Meta often
stores both decoded and still-encoded variants as **disjoint** session sets.
See [references/meta.md](references/meta.md).

### Conversion funnel

```sql
SELECT
  uniqIf(subject, name = 'replo.page_view') AS viewed,
  uniqIf(subject, name = 'replo.add_to_cart') AS added_to_cart,
  uniqIf(subject, name = 'replo.start_checkout') AS started_checkout,
  uniqIf(subject, name = 'replo.purchase') AS purchased
FROM events_computed FINAL
WHERE date >= '2026-03-19'
  AND date <= '2026-04-02'
```

### Top products by revenue (from purchase events)

```sql
SELECT
  JSONExtractString(line_item, 'variant', 'product', 'title') AS product_title,
  sum(JSONExtractFloat(line_item, 'finalLinePrice', 'amount')) AS total_revenue,
  sum(JSONExtractUInt(line_item, 'quantity')) AS total_quantity,
  count() AS order_count
FROM events_computed FINAL
ARRAY JOIN JSONExtract(data, 'payload', 'lineItems', 'Array(String)') AS line_item
WHERE name = 'replo.purchase'
  AND date >= '2026-03-19'
GROUP BY product_title
ORDER BY total_revenue DESC
LIMIT 10
```

### Sessions from a specific referrer

```sql
SELECT
  toDate(date) AS day,
  uniq(subject) AS sessions
FROM events_computed FINAL
WHERE name = 'replo.session_start'
  AND JSONExtractString(data, 'root', 'referrer') LIKE '%google.com%'
  AND date >= '2026-03-19'
GROUP BY day
ORDER BY day
```

## Important Notes

- **No namespace filter needed.** Queries are automatically scoped to the current project and any workspace siblings that share its Shopify store via ClickHouse row policies on every namespaced table. Adding `WHERE namespace = ...` is unnecessary; to isolate a single storefront, filter by `domain` (rows can span multiple sibling namespaces).
- **Prefer the `daily_*_rollups` tables** for date-range aggregates over the last 7+ days — they're dramatically faster than `events_computed`. Fall back to `events_computed FINAL` for sub-day granularity, the last 3 hours of data (rollups have a 3-hour lag), or dimensions the rollups don't carry.
- **Rollup `*_state` columns must be read with `*Merge`** combinators (`countMerge`, `uniqMerge`, `sumMerge`) and `GROUP BY` your dimensions. Selecting them raw returns binary state blobs.
- **Rollup revenue is already in USD** (converted via `convert_to_usd` at MV write time). Do not try to convert it again.
- **Use `WITH FILL STEP INTERVAL ...`** for time-series queries (daily / hourly / weekly grouping) so days with zero activity still appear as zero-valued rows instead of dropping out of the result. See "Filling gaps in time-series queries" under Common Query Patterns.
- **Don't `toString(...)` the time-bucket column when using `WITH FILL`** — the `FROM` / `TO` bounds are `Date` / `DateTime`, and a `String`-aliased column has no supertype with them. Keep the column native and format for display in your application code, not in SQL.
- **Be careful with `FINAL` on `events_computed`** — it's a ReplacingMergeTree on Cloud, so `FINAL` opens every active part in the partition range (high-latency S3 reads). Use `FINAL` only when your aggregates can be inflated by duplicates (`count()`, `sum()`); skip it for `uniq*`-only queries. When you do use `FINAL`, append `SETTINGS do_not_merge_across_partitions_select_final = 1` and never wrap `date` in `toDate(...)` — use a half-open range like `date >= toDate('X') AND date < toDate('Y') + INTERVAL 1 DAY` to preserve partition pruning. See the `events_computed` table notes above for details.
- **Only `SELECT` and `WITH` (CTE) queries are accepted.** `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `CREATE`, etc. will be rejected.
- **Server-enforced bounds:** queries are limited to **60 seconds of execution time** and **100,000 result rows**. Result-row overflow throws an error — narrow your `WHERE` clause, add `LIMIT`, or aggregate first if you hit the limit.
- Use materialized columns (`path`, `domain`, `subjectPurchaseSum`, etc.) instead of `JSONExtract` when possible — they're faster.
- For session-level purchase aggregates (total order value, purchase count), use the materialized columns on `events_computed FINAL`.
- For product-level purchase data (line items), `ARRAY JOIN` on `JSONExtract(data, 'payload', 'lineItems', 'Array(String)')` against `events_computed FINAL` filtered to `name = 'replo.purchase'`.

## Granularity Guidance

- For ranges up to ~7 days: group by hour (`toStartOfHour(date)`)
- For ranges up to ~6 months: group by day (`toDate(date)`)
- For ranges up to ~2 years: group by week (`toStartOfWeek(date)`)
- For multi-year ranges: group by month (`toStartOfMonth(date)`)
