# Intelligems live experiment audit

Run this on the **published build and actual experiment domains** when requested, after the [code audit](code-audit.md). This verifies delivered configuration, visitor journeys, Shopify checkout and Intelligems reporting. A code-only or localhost pass cannot satisfy these checks. Use an authorized test store/experiment; do not start merchant tests or place real paid orders without authorization.

Mark every applicable item passed, failed, blocked, or not applicable with a reason. Preserve URLs, SDK versions, experience/variation IDs, browser/time, sanitized network evidence and report screenshots. Never include API tokens or full checkout URLs/cart keys. A loaded SDK or accepted beacon alone does not prove attribution.

## Account and installation

- [ ] Correct store and Intelligems organization; headless storefront/domain configuration enabled for the intended host. External API credentials stay server-side; only the browser-safe Storefront token and organization ID enter the page bundle.
- [ ] On the published build, the integration details loader resolves the intended Shopify store and Intelligems makes a successful Storefront request using its public token. Missing Shopify setup or loader errors show usable default content without mounting Intelligems.
- [ ] List/get tools return the actual experience, its current status, targeting, distribution and variation IDs. Never guess IDs from labels. Creation tools only create a shell; verify routing/pricing configuration in Intelligems before starting it.
- [ ] One Intelligems runtime and one tracker per page tree. Check Scripts & Pixels, GTM, snippets and embedded scripts for duplicate standard Intelligems installations on Replo.
- [ ] If the site uses markets or multiple currencies, change markets without a full reload: cookie, refreshed server props, remounted Intelligems client, displayed price, cart and checkout currency agree. A prop change alone does not reconstruct headless 1.2.19's client.

## Assignment and rendering

- [ ] Both control and every treatment, first visit and return visit, direct entry and SPA navigation, mobile and desktop. Verify the rendered content agrees with the assignment in telemetry; also exercise ineligible, paused/ended/missing experiences and no active experiment.
- [ ] Wait for relevant `useIgVariation` readiness. Config downloaded is not proof that asynchronous targeting has resolved. An assigned control is measured; a fallback rendered because assignment failed is unmeasured.
- [ ] Block/delay the SDK/config endpoint. The rest of the page remains usable while assignment loads, and thrown errors show usable default content. Tracking stays gated on assignment readiness and the content actually displayed. Test late recovery as well as immediate rejection; do not record treatment while default fallback content stays visible.
- [ ] Preview/force links select each intended cohort, and a clean profile without overrides produces normal measured traffic. Clear cookies **and** storage between fresh-visitor cases. Retain them for sticky-assignment tests.
- [ ] Normal reporting tests use a fresh non-preview cart. A cart created in preview can retain `igPreview=true` after ordinary tracking updates its visitor and group attributes.
- [ ] Inspect the config actually received by the published page after starting a test or changing domains. Account settings and cached CDN configuration can disagree; verify status and domain before diagnosing assignment or cookies.

## Redirects and domains

- [ ] Test both origin-to-control and origin-to-Replo-treatment through the actual entry URL, both on direct load and after Next.js Link/router navigation from another page. Test Replo-as-origin separately if required. Confirm one redirect, correct final URL, query preservation, no loop and no redirect on excluded paths.
- [ ] On sibling domains, inspect the actual cookie Domain/Path/Secure settings and verify identical visitor identity and variation at both ends. Sharing a registered domain makes sharing possible; it does not prove the installed clients use root-domain cookies. Host-only cookies and localStorage do not span siblings.
- [ ] For shared-domain journeys, verify the [required patch or verified vendor fix](package-patch.md) is present in the published build. Test fresh visitors and existing host-scoped cookies/localStorage for identity continuity; preserve an existing shared identity during migration. Record the exact package/patch version.
- [ ] Direct destination visits and a returning visitor re-entering the origin have an explicit, verified behavior. Do not assume arrival at a variation URL assigns its cohort.
- [ ] Unrelated registered domains are not supported by the standard shared-cookie setup. Do not claim `replosites.com` preserves assignment.

## Consent and telemetry

The first two checks apply if the site uses a cookie consent setup. Check this first; telemetry and goals still apply without a consent UI.

- [ ] Before hydration/consent resolution, mount no Intelligems provider/tracker. Declined consent renders usable default content without SDK tracking, cookies or attributed cart writes. Test initial unknown, accept, decline, revoke and navigation after each state using the site's actual CMP.
- [ ] Revocation unmounts the runtime and stops custom-event handlers; no queued goals replay on reacceptance. Inspect network activity and existing attributed carts. If policy requires removing existing identifiers/attributes, implement and verify that cleanup too; unmounting does not erase stored attribution or cancel requests already in flight.
- [ ] One owner of native page views. Inspect visitor ID, variation IDs, currency, country and preview/exclusion state at the ingestion boundary. `useIgTrack` may emit sitewide events with no active experiment; it is not an automatic experiment-only consent gate.
- [ ] [Custom goals](https://headless.intelligems.io/examples/custom-events) have stable names and identical semantics across cohorts. Quiz milestones fire from successful actions, with duplicate submission protection and no personal answers. No blanket click mapping. Verify custom events obey consent and assignment readiness.
- [ ] Custom metric display name and emitted event identifier are matched. For a goal created with Define an event, use its generated identifier, remove unused placeholder listener code, and select the metric before starting the test.
- [ ] Where GA4 is used, verify the configured measurement ID and Intelligems' GA4 integration against the store's actual setup. Check GA4 separately from native Intelligems goals and order attribution; a GA4 event does not prove an Intelligems result, or vice versa.

## Cart, Buy Now, pricing, shipping and offers

- [ ] Regular cart: verify the cart ID sent by Intelligems matches the current Replo cart. Test no cart → cart creation, adding/removing lines, returning cart and cart replacement. Verify real Shopify cart attributes contain the visitor's assignment before checkout.
- [ ] Immediate checkout after assignment/cart creation is covered. `useIgCart` performs asynchronous writes; hook readiness is not a documented write-completion promise. Do not claim a race is solved by waiting a fixed delay. If a reliable persistence barrier is absent, record the regular-cart path as blocked until implemented/verified.
- [ ] Buy Now: create its separate cart with exact `useIgCartAttributes().attributes` after readiness. Verify attributes in `cartCreate` and on the resulting order. Test immediate click, repeated click, request failure, quantity, selected variant and selling plan. An anchor/permalink to checkout does not gain these attributes merely by loading the tracker.
- [ ] Price test: exercise the selected Shopify variant and currency, plus each supported selling plan through its configured subscription integration. See [price hooks](https://headless.intelligems.io/reference/hooks/price-hooks) and [Shopify Functions integration](https://headless.intelligems.io/usage/update-atc-events/shopify-functions). Verify control/treatment display prices, compare-at prices, cart totals, discounts and final charged amount. Changing displayed text alone is not a price integration.
- [ ] Shipping test: verify the required wrapped line attributes from `useIgCart().wrapCustomAttributes` survive both regular-cart and Buy Now creation and produce the assigned shipping rate in checkout. Verify the Replo purchase API supports each required line property; do not assume cart-level attributes replace line attributes.
- [ ] Offer/discount test: verify Shopify Functions/discount configuration, eligibility, stacking, subscription behavior and final order values in every cohort. Unsupported combinations are explicitly excluded from launch claims.

## Reporting proof and cleanup

- [ ] Use a designated test store/experiment. Finish a test order for each applicable checkout path and cohort. Record sanitized order IDs, expected amounts/currency, visitor/variation and time. Verify those orders are included in Intelligems' report under the correct variation; test orders may be filtered, so confirm report inclusion rules first.
- [ ] Verify named custom goals and visitor/conversion/revenue results after the documented processing delay, with the intended date range, timezone, preview and audience filters. An accepted beacon only proves transport, not dashboard processing. Record any filtering/delay as unresolved until results are observed.
- [ ] Start or resume the test **before** creating the clean measurement session. Intelligems' always-on filters exclude sessions begun before the test started or while it was paused, and visitors who previewed the test. Removing `igPreview` from a cart or enabling test-order reporting does not reverse those session exclusions. Keep preview QA and measured orders in separate fresh browser profiles.
- [ ] For a missing order, inspect the vendor's Order Reconciliation export when available; it reports exclusion reasons and may require account enablement. Otherwise repeat with a fresh post-start session and record the actual page-view identity, assignment/exclusion flags and order identity. Do not call a missing report row a transport bug solely because the order contains `igTestGroups`. See [default filters](https://docs.intelligems.io/analytics/overview/experiment-analytics/filters) and [session attribution](https://docs.intelligems.io/analytics/overview/how-orders-and-sessions-are-attributed-to-experiments).
- [ ] For refunds/cancellations/discounts, check the selected revenue metric's definition before comparing values. Do not substitute Replo analytics for Intelligems totals.
- [ ] Pause/end only the dedicated tests created for verification, restore temporary configuration, remove test-only debug overrides and document any retained fixture site. Never alter unrelated merchant experiments to manufacture evidence.

Report exactly which permutations passed and which are blocked. Do not say “all experimentation works” when shipping, pricing, cross-domain identity, consent cleanup or order reporting was not exercised.
