# Revenue and attribution

Use this reference whenever a query includes revenue, purchases, conversion rate, AOV, ROAS, or revenue per session. Decide the business meaning before choosing columns. Ask whether the user wants overlapping session revenue or additive page credit when that choice is unclear. Do not ask again when already specified; label the chosen model visibly.

## Choose and label the metric

| Metric | Meaning | Domain selection | Can page rows be added? |
| --- | --- | --- | --- |
| Session revenue (USD) | Full purchase value of sessions that touched the selection in the report window | Select sessions by their matching events, then include their purchases across authorized sibling projects | No. A session can touch several pages; deduplicate by `subject` for a combined total |
| Attributed revenue (USD; equal pageview credit) | Session purchase value divided equally among eligible pageviews in the report window | Filter credited pageviews **after** calculating shares across all authorized domains | Yes, for disjoint sets of eligible pageviews; checkout/cart receive no credit |
| Purchase revenue on selected domains (USD) | Purchases whose purchase event occurred on the selection | Filter the purchase events themselves | Yes for disjoint purchase events, but it does not measure landing-page contribution |

A $48 purchase after one landing view in project A and a checkout view in linked project B gives landing attributed=$48/session=$48; checkout attributed=$0/session=$48. Selecting both gives **$48 total session revenue, never $96**. With two eligible pageviews, each gets $24 attributed revenue even when selected alone. Repeated views of a page each get a share: this is equal **pageview** credit, not equal unique-page, first-touch, last-touch, or causal lift.

Do not label these different calculations simply "Revenue" without a visible definition. Do not invent first/last-touch support or a marketing attribution window. If the requested model differs, agree on its identity, eligible touches and window, and validate the custom calculation separately.

## Shared calculation

Use `session_attribution(since='YYYY-MM-DD', until='YYYY-MM-DD')` for cross-project session and page attribution. `since` and `until` are required, inclusive **UTC dates**. Use literal dates in the queries below; replace the example dates and domain with the requested scope. Do not add `FINAL` to the view.

The server scopes source events to the current project and authorized same-workspace, same-Shopify-store siblings. The view deduplicates source rows, joins by session UUID (`subject`) across that authorized set, and computes totals **before** caller domain/path/UTM filters. Never supply another project's namespace or broaden authorization yourself.

This is **report-window attribution**: only events and purchases inside the supplied dates participate. A landing view before `since` does not earn credit for a purchase inside the window; a purchase after `until` is not included. Changing dates can change shares. Changing only selected domains/pages must not. Compute comparison periods with separate view calls, not a combined current+previous window. Label this window in the report; do not describe these values as lifetime session revenue.

The view returns `namespace, subject, date, id, name, data, domain, path, title` and:

| Column | Meaning |
| --- | --- |
| `purchaseRevenueUsd` | Purchase event's converted value; zero on other events |
| `sessionRevenueUsd` | All purchases in the session within this report window, repeated on every event |
| `sessionPurchaseCount` | Number of purchase events within this report window, repeated on every event |
| `attributedRevenueUsd` | Session revenue divided by the number of eligible pageviews; zero on non-pageviews and paths containing `/checkout` or `/cart` |

Currency conversion uses the purchase date's exchange rate, then the latest available rate, then 1:1 when no rate exists (the existing `convert_to_usd` policy). Missing currency/rates are a reporting limitation: inspect unexpected currencies and disclose gaps rather than presenting unverified conversion as exact. Purchases without an eligible pageview have session revenue but no attributed page revenue. Pixel totals need not equal Shopify's ledger (tracking coverage, refunds and order adjustments differ).

**Do not use `subjectPurchase*` or page/conversion rollups for cross-project attribution.** Those stored fields are computed within `(namespace, subject)` and can be zero on the landing project's events when checkout is in a sibling. Taking `max`/`any` of those values after selecting a landing domain does not repair the missing purchase. Rollups remain useful for traffic or explicitly project-local analysis, with their 3-hour lag.

## Canonical queries

### Session revenue total: select membership, deduplicate once

```sql
SELECT sum(revenue) AS session_revenue_usd
FROM (
  SELECT subject, max(sessionRevenueUsd) AS revenue
  FROM session_attribution(since='2026-07-01', until='2026-07-30')
  WHERE domain = 'landing.example.com'
  GROUP BY subject
)
```

For a page table, group the inner query by **domain, path, subject**, then sum per domain/path. Recompute the grand total by subject across the whole selection; never sum the page rows. Filtering to `name = 'replo.page_view'` deliberately requires a recorded pageview for membership.

### Attributed revenue by page

```sql
SELECT domain, path, sum(attributedRevenueUsd) AS attributed_revenue_usd,
       uniqExact(subject) AS sessions,
       uniqExactIf(subject, sessionPurchaseCount > 0) AS purchasing_sessions,
       purchasing_sessions / nullIf(sessions, 0) AS conversion_rate
FROM session_attribution(since='2026-07-01', until='2026-07-30')
WHERE name = 'replo.page_view' AND domain = 'landing.example.com'
GROUP BY domain, path
ORDER BY attributed_revenue_usd DESC
LIMIT 100
```

Do not recalculate the denominator in this outer query. Do not sum fractional amounts on purchase events; credit is on eligible pageview rows. Conversion rate uses distinct purchasing sessions / distinct sessions, not fractional purchase credits.

### Purchase-date session revenue trend / AOV

```sql
WITH scoped_sessions AS (
  SELECT DISTINCT subject
  FROM session_attribution(since='2026-07-01', until='2026-07-30')
  WHERE domain = 'landing.example.com'
)
SELECT toDate(date) AS day,
       sum(purchaseRevenueUsd) AS session_revenue_usd,
       count() AS purchases,
       session_revenue_usd / nullIf(purchases, 0) AS average_order_value_usd
FROM session_attribution(since='2026-07-01', until='2026-07-30')
WHERE name = 'replo.purchase' AND subject IN (SELECT subject FROM scoped_sessions)
GROUP BY day
ORDER BY day
```

Scope belongs in the membership query, **not** on the outer purchase rows. For mixed-provider queries, apply the domain filter only to Replo events. Fill missing dates for complete time axes. AOV divides by purchases, not purchasing sessions; revenue per session divides the deduplicated total by the matching session cohort. Keep the numerator and denominator on the same date range and membership definition.

## Verify the meaning, not just SQL syntax

Run the queries through `query_replo_analytics` using literal dates/filters before saving. Check one converting session across landing/checkout domains; compare landing only, checkout only, and both; inspect zero-result and no-purchase cases. Compare total session revenue with a deduplicated raw-purchase query over the same cohort. Attributed totals reconcile only for purchases with eligible pageviews. Do not claim observed checks passed if the project has no matching example; report that coverage gap and use the shared calculation.
