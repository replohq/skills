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

## Resolve metric meaning before choosing SQL

A familiar metric name is not a complete definition. Use the request, prior conversation, existing report definitions and available data to determine what the number should represent. Before choosing columns, check the choices that could materially change the result:

- **What is counted:** events, sessions, distinct people, orders or another entity; which event or condition qualifies; which identity supports deduplication.
- **Who is included:** the population, domains/pages, exclusions, and whether filters select matching events or sessions/entities that touched the selection.
- **How it is calculated:** numerator and denominator for rates/averages, grouping and time buckets, whether groups overlap, and whether a total can be summed or must be recomputed.
- **When and from where:** reporting/comparison dates, timezone, event date versus cohort date, source and freshness, and units or currency where relevant.

Treat these as reasoning checks, not a questionnaire. If two plausible interpretations would materially change the answer and context does not resolve them, ask a concise question in business language and explain the difference. Ask 1–3 short questions at a time, each resolving one main choice; do not hide a long questionnaire in compound questions. Prioritize choices that change the business conclusion. Reuse established reporting defaults for incidental settings and state them instead of asking about every possible option. Continue independent data discovery while waiting, but do not silently choose an unresolved definition. If the request is clear, use it; if the user delegates choices, state a suitable default and proceed. Never promise a definition the available tracking cannot support.

Examples of ambiguity include "traffic" (pageviews, sessions or identifiable visitors), "conversion rate" (sessions with at least one purchase / sessions versus purchase events / sessions), and "average engagement" (which event or duration, averaged over which population). Revenue attribution is another instance of the same problem; follow the reference below for its specific rules. Do not introduce ambiguity the user has already resolved: "daily distinct sessions for the last 30 UTC days on this domain" needs no metric-definition question.

Confirm source field paths, types and units against the schema or representative events before relying on them. Keep the chosen definition consistent across SQL, labels, filters and totals. Name the counted entity or ratio clearly; disclose material scope, window, overlap and data limitations alongside the result. Validate using the same population and definition: deduplicate distinct counts across overlapping groups, recompute overall rates from the underlying population rather than averaging row percentages or summing overlapping groups, and distinguish missing data from a measured zero.

## Revenue, purchases, and conversion metrics

**Required:** Read [references/revenue.md](references/revenue.md) before writing a query with revenue, purchases, conversion rate, AOV or ROAS. It defines the metrics and the shared `session_attribution(since, until)` calculation. Stored `subjectPurchase*` fields and page/conversion rollups are **project-local**, even though queries can read linked projects. Domain filtering purchase rows measures checkout location, not landing-page contribution.

## Project Scoping (Important)

Queries are **automatically scoped to the current project and any sibling projects in the same workspace that share its Shopify store** by the server. Every query is executed with a per-project ClickHouse role, and the namespaced tables (`events_computed`, `namespace_to_domain`, `daily_page_rollups`, `daily_namespace_rollups`, `daily_namespace_purchase_rollups`) all have RESTRICTIVE row policies that filter rows to those namespaces before they reach you. **You do not need to add `WHERE namespace = ...` filters to your queries.**

Because Shopify's web pixel stamps every conversion event with the single project that first activated the pixel on a store, a store's analytics can live under a sibling project's namespace. Scoping to every workspace sibling that connects the same Shopify store makes that shared-store data visible from any of them. Practically this means: rows can carry **more than one `namespace`**, and `namespace_to_domain` can map to **multiple domains** — so filter/group by `domain` (or `namespace`) when you need to isolate a single storefront.

## Tables

The agent has SELECT access to the following tables and `session_attribution` view (plus optional external-integration metric view functions — see below). Anything else will fail with a permission error:

| Table                              | Description                                                                                                                                                                                              | Row-policied?                |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `session_attribution(since, until)` | Shared report-window session and page revenue across authorized sibling projects; see the required revenue reference | Inherits source row policies |
| `events_computed`                  | Session-level computed analytics (PRIMARY TABLE FOR AD-HOC QUERIES)                                                                                                                                      | Yes — scoped to your project |
| `namespace_to_domain`              | Maps namespace → domain. Useful for resolving domain names.                                                                                                                                              | Yes — scoped to your project |
| `currency_exchange_rates`          | Currency exchange rates for multi-currency support.                                                                                                                                                      | No (global)                  |
| `daily_page_rollups`               | Pre-aggregated daily metrics per (day, domain, path, utm_source). USD-converted. **Prefer this for date-range page analytics over `events_computed`.**                                                   | Yes — scoped to your project |
| `daily_namespace_rollups`          | Pre-aggregated daily traffic metrics per (day, utm_source). Views, sessions, converting sessions.                                                                                                        | Yes — scoped to your project |
| `daily_namespace_purchase_rollups` | Pre-aggregated daily purchase metrics per (day, utm_source). Purchase count, USD revenue, unique purchasing sessions.                                                                                    | Yes — scoped to your project |
| `consent_events`                   | Proof-of-consent audit log: one append-only row per visitor consent decision (`action`, `consent_mode`, `policy_version`, `categories` JSON, `decided` via `date`). Use for GDPR consent exports/audits. | Yes — scoped to your project |

### When to use rollups vs `events_computed`

The three `daily_*_rollups` tables are **`AggregatingMergeTree`** tables refreshed every 5 minutes by background materialized views, with a **3-hour data lag** (the most recent 3 hours of data is not yet in the rollups). They are dramatically faster than `events_computed` for any query that fits their grain (daily, by namespace + optional domain/path/utm_source) and is looking for the data that they roll-up over.

**Prefer rollups for traffic or explicitly project-local metrics. For revenue/conversion analysis spanning linked projects, use `session_attribution` instead.**

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

**These rollup revenue/conversion fields are project-local.** They do not join a landing session to a purchase in a sibling namespace. For cross-project reporting use `session_attribution`; see the revenue reference for additive page credit versus overlapping session totals.

### `daily_namespace_rollups` — Per-day traffic aggregates

`AggregatingMergeTree` keyed on `(day, namespace, utm_source)`. Traffic and project-local converting sessions (no purchase count/revenue); join with `daily_namespace_purchase_rollups` for purchase metrics.

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

This is a `ReplacingMergeTree` table keyed on `(namespace, subject, date, id)` with `insertDate` as the replacement version column. The flusher computes session-level aggregates (purchases, attribution) and writes them into the `computed` JSON column.

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
| `computedVersion` | UInt8         | Computation algorithm version                      |
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
- `data.payload.currencyCode` — Currency code (e.g. `USD`)
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
| `subjectPurchaseSum`           | Float64 | `computed.subject.purchase.sum`           | Total purchase amount for this **project/session**                   |
| `subjectPurchaseCount`         | Float64 | `computed.subject.purchase.count`         | Number of purchases in this **project/session**                      |
| `subjectPurchaseFraction`      | Float64 | `computed.subject.purchase.fraction`      | Fractional attribution weight for this **project/session**           |
| `subjectPurchaseFractionalSum` | Float64 | `computed.subject.purchase.fractionalSum` | Fractionally attributed purchase amount for this **project/session** |

**The `subject*` columns are legacy project-local aggregates, repeated on event rows.** They group `(namespace, subject)`, not the full authorized cross-project session. Summing repeated totals overcounts, and deduplicating a zero landing-project total cannot recover sibling checkout revenue. For revenue and conversion reporting use `session_attribution` and the recipes in [references/revenue.md](references/revenue.md).

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

Connected providers include **Triple Whale**, **Contentsquare**, and **Intelligems**. Discover available view names and metrics from the project's analytics schema before querying; do not assume a provider is connected.

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
WHERE day >= today() - 29
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
WHERE day >= today() - 29
GROUP BY day
ORDER BY day
```

#### Top revenue-driving pages / UTM sources

Use the attributed-page query in [references/revenue.md](references/revenue.md). For UTM attribution, group eligible pageview credits by `JSONExtractString(data, 'root', 'params', 'utm_source')` instead of page. This gives equal pageview credit by observed UTM, not first/last-touch marketing attribution. Use session membership if the user instead wants all revenue from sessions touching a source.

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
WHERE day >= today() - 29
GROUP BY day
ORDER BY day ASC
WITH FILL
  FROM today() - 29
  TO today() + 1
  STEP INTERVAL 1 DAY
```

Without `WITH FILL`, a 30-day query that has data on only 18 days returns 18 rows. With `WITH FILL`, you get all 30 rows (including the start and end-of-range days), with `views = 0` / `sessions = 0` on the days where nothing happened.

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
WHERE day >= today() - 29
GROUP BY day
ORDER BY day ASC
WITH FILL
  FROM today() - 29
  TO today() + 1
  STEP INTERVAL 1 DAY
INTERPOLATE (cumulative_revenue_usd)
```

### Purchase metrics, AOV, page conversions, and UTM attribution

Use the shared view and canonical queries in [references/revenue.md](references/revenue.md). AOV is revenue / purchase count, not revenue / purchasing-session count. Never sum repeated `subjectPurchaseSum` values on pageviews. For UTM groups, apply the filter after the shared calculation; recompute session-revenue totals by distinct subject instead of adding overlapping groups.

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
  AND date < toDate('2026-04-02') + INTERVAL 1 DAY
```

### Top products by revenue (from purchase events)

```sql
SELECT
  JSONExtractString(line_item, 'variant', 'product', 'title') AS product_title,
  JSONExtractString(data, 'payload', 'currencyCode') AS currency,
  sum(JSONExtractFloat(line_item, 'finalLinePrice', 'amount')) AS total_revenue,
  sum(JSONExtractUInt(line_item, 'quantity')) AS total_quantity,
  uniqExact(id) AS order_count
FROM events_computed FINAL
ARRAY JOIN JSONExtract(data, 'payload', 'lineItems', 'Array(String)') AS line_item
WHERE name = 'replo.purchase'
  AND date >= '2026-03-19'
GROUP BY product_title, currency
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
- **Prefer the `daily_*_rollups` tables** for traffic or explicitly project-local aggregates over the last 7+ days — they're dramatically faster than `events_computed`. Fall back to `events_computed FINAL` for sub-day granularity, the last 3 hours of data (rollups have a 3-hour lag), or dimensions the rollups don't carry.
- **Rollup `*_state` columns must be read with `*Merge`** combinators (`countMerge`, `uniqMerge`, `sumMerge`) and `GROUP BY` your dimensions. Selecting them raw returns binary state blobs.
- **Rollup revenue is already in USD** (converted via `convert_to_usd` at MV write time). Do not try to convert it again.
- **For Insights dashboard trends, use the bucket spine required by `insights-dashboard`. For other time-series queries, use `WITH FILL STEP INTERVAL ...`** (daily / hourly / weekly grouping) so days with zero activity still appear as zero-valued rows instead of dropping out of the result. See "Filling gaps in time-series queries" under Common Query Patterns.
- **Don't `toString(...)` the time-bucket column when using `WITH FILL`** — the `FROM` / `TO` bounds are `Date` / `DateTime`, and a `String`-aliased column has no supertype with them. Keep the column native and format for display in your application code, not in SQL.
- **Be careful with `FINAL` on `events_computed`** — it's a ReplacingMergeTree on Cloud, so `FINAL` opens every active part in the partition range (high-latency S3 reads). Use `FINAL` only when your aggregates can be inflated by duplicates (`count()`, `sum()`); skip it for `uniq*`-only queries. When you do use `FINAL`, append `SETTINGS do_not_merge_across_partitions_select_final = 1` and never wrap `date` in `toDate(...)` — use a half-open range like `date >= toDate('X') AND date < toDate('Y') + INTERVAL 1 DAY` to preserve partition pruning. See the `events_computed` table notes above for details.
- **Only `SELECT` and `WITH` (CTE) queries are accepted.** `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `CREATE`, etc. will be rejected.
- **Server-enforced bounds:** queries are limited to **60 seconds of execution time** and **100,000 result rows**. Result-row overflow throws an error — narrow your `WHERE` clause, add `LIMIT`, or aggregate first if you hit the limit.
- Use materialized columns (`path`, `domain`, etc.) instead of `JSONExtract` when possible — they're faster.
- For session-level purchase aggregates and page attribution, use `session_attribution(since, until)`; read the revenue reference first.
- For product-level purchase data (line items), `ARRAY JOIN` on `JSONExtract(data, 'payload', 'lineItems', 'Array(String)')` against `events_computed FINAL` filtered to `name = 'replo.purchase'`.

## Granularity Guidance

- For ranges up to ~7 days: group by hour (`toStartOfHour(date)`)
- For ranges up to ~6 months: group by day (`toDate(date)`)
- For ranges up to ~2 years: group by week (`toStartOfWeek(date)`)
- For multi-year ranges: group by month (`toStartOfMonth(date)`)
