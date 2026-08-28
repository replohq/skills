# Meta Ads and Replo Analytics

Use this reference when a user asks about **Meta Ads Manager / Events Manager**
numbers (sessions, adds to cart, checkouts, purchases), and optionally how they compare to what you get from
`query_replo_analytics`. If there are discrepancies, treat a raw
integer mismatch as a **definition / attribution / matching** problem first —
not proof that the Replo pixel is broken.

## What Replo can and cannot see

| Surface | What it is | Agent access |
| --- | --- | --- |
| Replo first-party (`replo.*` in ClickHouse) | Browser events from the Replo storefront pixel | Yes — `query_replo_analytics` |
| Meta Pixel on the Replo site (`fbq`) | Outbound browser events to Meta | Install/configure via tracking-scripts; **no Ads Insights pull** |
| Meta Ads Manager | Ad-attributed results (pixel + CAPI + modeled), inside an attribution window | **No API** — only what the user pastes / screenshots |
| Shopify → Meta (native channel / CAPI) | Server-side checkout & purchase for Shopify-hosted checkout | Not in Replo analytics |

There is **no Meta Marketing API / Ads Insights integration**. You cannot fetch
Ads Manager “Website adds to cart” from tools. You can only query Replo
first-party (and other third-party insights sources, such as Triplewhale, Northbeam, etc) from analytics tools.

Outbound Meta standard events from `MetaSink`:

| Replo sink | Meta `fbq` event |
| --- | --- |
| `pageView` | `PageView` |
| `productViewed` | `ViewContent` |
| `addToCart` | `AddToCart` |
| `startCheckout` | `InitiateCheckout` |
| `purchase` | `Purchase` |

## Shopify checkout gate (why Meta checkouts can exist while Replo Meta never fires them)

When the user is using Shopify checkout, third-party ad sinks (including Meta) are
wrapped with `gateShopifyOwnedFunnelEvents`. That **no-ops** `startCheckout` and
`purchase` on the storefront pixel so Shopify’s native Meta channel / CAPI owns
those events (no shared `event_id` across origins to dedupe).

- **Still fires from Replo → Meta:** `PageView`, `ViewContent`, `AddToCart`
- **Suppressed from Replo → Meta:** `InitiateCheckout`, `Purchase`
- **Still recorded in Replo first-party:** `replo.start_checkout` / `replo.purchase` when the storefront emits them (purchases on Shopify thank-you often never hit Replo — expect `replo.purchase = 0` on many Shopify-checkout agent sites)

So: Ads Manager can show website checkouts/purchases from **Shopify → Meta**
while a Replo Meta pixel test on the lander never sees `InitiateCheckout` /
`Purchase`. That is intentional, not a missing lander script.

## Never compare Ads Manager to UTM-filtered Replo without labeling it

**Meta Ads Manager “Website adds to cart / checkouts / purchases”** =

- Events Meta can **attribute** to the ad (default window roughly **7-day click +
  1-day view**, plus engage-through variants Meta has been changing in 2026)
- From **any domain** allowed on that pixel dataset (brand main site, `shop.app`,
  account subdomain, Replo custom domain, etc.)
- Pixel **and** Conversions API, after Meta’s matching / dedupe / modeling

**Replo `query_replo_analytics` for an ad set** (typical agent query) =

- First-party `replo.*` rows only
- Usually filtered by **exact** `utm_*` (or path) on the landing session
- **No** view-through: if they saw the ad and later converted on
  `www.brand.com` without the campaign UTM, Meta may count it; Replo UTM match
  will not

A large gap in e.g. Meta add-to-carts vs Replo add-to-carts is **suspicious and
worth investigating**, but it is **not automatically a Replo issue**.
Common real causes, in order of how often they show up:

1. **Attribution scope** — Ads Manager includes view-through + other domains on
   the same pixel; Replo query is UTM/path-scoped to the Replo lander.
2. **UTM encoding mismatch** — agent matched only one spelling of `utm_term`
   (see below) and under-counted sessions (does not by itself invent Meta ATCs).
3. **Pixel + CAPI without shared `event_id`** — Replo’s `MetaSink` does **not**
   currently send Meta `eventID`. If Shopify (or another CAPI source) also sends
   `AddToCart` for the same action without the same id, Ads Manager / Events
   Manager can inflate vs first-party. Confirm with a live `fbq` + network test
   before asserting this.
4. **On-page manual tracking double `fbq('track','AddToCart')`** — rare; verify with the
   procedure below before blaming the lander.

If comparing Replo vs Meta events to the user, say explicitly: “Replo first-party” vs
“Meta attributed website conversions,” and give both numbers.

## UTM matching pitfalls (Meta ad set → Replo sessions)

Meta URL tags often land in ClickHouse **both decoded and still-percent-encoded**.
These can be **disjoint session sets** (zero overlap):

| Stored `utm_term` example | Notes |
| --- | --- |
| `JDH129_MW-18+_CPR-PUR-Max-Conv_url=all-bundles-lander` | Decoded `+` and `=` |
| `JDH129_MW-18%2B_CPR-PUR-Max-Conv_url%3Dall-bundles-lander` | Literal `%2B` / `%3D` in the string |
| `JDH129_MW-18 _CPR-PUR-Max-Conv_url=all-bundles-lander` | `+` turned into space |

**Do not** equality-match a screenshot, pasted ad set name, etc to `utm_term` alone. Prefer normalizing before compare:

```sql
WITH
  normalize_utm AS (
    SELECT
      subject,
      replaceAll(
        replaceAll(
          replaceAll(
            JSONExtractString(data, 'root', 'params', 'utm_term'),
            '%2B',
            '+'
          ),
          '%3D',
          '='
        ),
        ' ',
        '+'
      ) AS utm_term_norm
    FROM events_computed FINAL
    WHERE date >= today() - 30
  )
SELECT ...
WHERE utm_term_norm = 'JDH129_MW-18+_CPR-PUR-Max-Conv_url=all-bundles-lander'
```

Also match on **campaign** when the ad set name is truncated in the UI:

```sql
JSONExtractString(data, 'root', 'params', 'utm_campaign') = '260817_Bundles_CPR-ASC_JDH'
```

And remember: UTMs usually exist on the **landing `replo.page_view`**, not on
every later event. Attribute the session with a `subject IN (SELECT DISTINCT
subject ... WHERE utm_... = ...)` pattern, then count funnel events for those
subjects (any path).

## Funnel event names (always query checkout, not only purchase)

| Question | Event |
| --- | --- |
| Add to cart | `replo.add_to_cart` |
| Checkout started | `replo.start_checkout` |
| Purchase | `replo.purchase` |

Product fields for ATC live under **`data.payload`**, not `data.root`:

```sql
JSONExtractString(data, 'payload', 'productTitle') AS product_title,
JSONExtractFloat(data, 'payload', 'price') AS price
```

## Live check: is the lander double-firing Meta AddToCart?

Run this on the published URL before telling the user “we’re double-counting”. Do NOT add this to the user's site source code.

1. Open DevTools → Console. Install a wrapper:

```js
window.__atcLog = [];
(function () {
  const orig = window.fbq;
  window.fbq = function () {
    window.__atcLog.push({ t: Date.now(), args: Array.from(arguments) });
    console.log("FBQ", window.__atcLog.length, arguments[0], arguments[1]);
    if (typeof orig === "function") return orig.apply(this, arguments);
  };
  if (orig) {
    ["queue", "loaded", "version", "callMethod"].forEach((k) => {
      try {
        window.fbq[k] = orig[k];
      } catch (_) {}
    });
  }
})();
window.__atcLog = [];
```

2. Network → clear → filter `facebook` or `tr/?id=`.
3. Click **one** Add to Cart.
4. Expect:
   - `window.__atcLog.filter((x) => x.args[1] === "AddToCart").length === 1`
   - One logical AddToCart (Meta may show more than one `facebook.com/tr` row for
     transport of the same `fbq` call — count **`fbq('track','AddToCart')`**, not
     raw request rows)
5. Confirm the configured pixel id appears once in scripts (e.g.
   `487093728122476`), not two different Meta identifiers.

If step 4 is `1`, the lander is **not** application-level double-firing Meta ATC.
Look at attribution scope, UTM matching, and Pixel vs Shopify CAPI next.

## Cross-domain Shopify checkout handoff (large Ads Manager gaps)

Agent Shopify-checkout sites often land ads on a **Replo custom domain**
(e.g. `caf.brand.com`) and send Checkout to the **merchant Shopify domain**
(e.g. `www.brand.com/checkouts/...`).

Observed pattern (a bundles lander on a Replo custom domain):

1. Ad → `caf.brand.com/all-bundles-lander?...&fbclid=...`
2. Replo records `replo.page_view` / `replo.add_to_cart` / sometimes
   `replo.start_checkout` on **caf only**
3. Cart Checkout navigates to
   `https://www.brand.com/checkouts/cn/...?fbclid=...&utm_...=...`
4. Same Meta pixel dataset also receives events from `www`, `shop.app`, etc.
   (Events Manager → Trafic / domains)
5. Shopify’s Meta channel / CAPI on **www checkout** can report
   `InitiateCheckout` / `Purchase` into Ads Manager while Replo’s Meta sink
   never fires those (Shopify gate) and Replo first-party never sees www events

**Product impact:** Ads Manager “Website checkouts” can be ~10× Replo
`replo.start_checkout` for the same ad set without any lander double-fire.
That is usually **attribution + domain split**, not a broken AddToCart button.

For ATC specifically: if Ads Manager ≫ Replo `replo.add_to_cart` after the
live `fbq` test shows 1:1 on the lander, ask for Events Manager **AddToCart
breakdown by domain** (or URL). Extra volume on `www` / `shop.app` means Meta
is counting off-Replo ATCs attributed to the ad (click/view window), which
Replo UTM queries on the lander will never include.

Also compare Meta **link clicks** (not “Clicks (all)”) to Replo sessions —
“Clicks (all)” is broader and often much higher than lander sessions.

Param survival: `useBuyNow` copies current URL search params (including
`fbclid` / UTMs) onto non-Stripe checkout URLs. Plain `<a href={checkoutUrl}>`
carts may or may not; verify the live checkout URL bar before assuming Meta
lost the click id.

## Response checklist for Meta discrepancy threads

1. Restate both numbers with **source labels** (Ads Manager attributed vs Replo
   first-party).
2. Re-query with **normalized UTMs** + campaign id; report session counts for
   decoded-only, encoded-only, and union.
3. Funnel: `page_view` / `add_to_cart` / `start_checkout` / `purchase` (do not
   skip checkout).
4. Note Shopify gate if `checkoutProvider` is Shopify.
5. Check whether Checkout leaves the Replo domain for `www` / Shopify
   checkout; if yes, treat Ads Manager checkouts/purchases as mostly
   **Shopify-domain** events.
6. Only claim on-page double-count after the live `fbq` test above.
7. Do not invent Meta Ads Insights queries — we do not have that API.
   Prefer asking the user for Events Manager domain breakdown when ATC still
   diverges after (5)–(6).
