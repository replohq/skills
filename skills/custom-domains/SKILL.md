---
name: custom-domains
title: Connect a Custom Domain
summary: Point your own domain at a Replo site, with the DNS records it needs.
description: "Use when a user asks to connect, add, verify, check, or troubleshoot a custom domain, configure DNS records, set up a root/apex/naked domain, or set up a subdomain. Also use when DNS verification fails, a domain is stuck pending, the user needs to see what DNS records to configure for an existing domain, or a domain shows \"refused to connect\" or redirects unexpectedly. Triggers: \"connect my domain\", \"add a custom domain\", \"set up example.com\", \"DNS records\", \"domain not working\", \"domain pending\", \"domain stale\", \"verify my domain\", \"activate domain\", \"what DNS records do I need\", \"domain stuck\", \"domain not verifying\", \"refused to connect\", \"www not working\", \"domain redirecting\"."
tools: connect_custom_domain, get_custom_domain_status, verify_custom_domain, list_sites
---

# Custom Domains

Connect a custom domain to a Replo site. Replo handles DNS verification, Cloudflare hostname provisioning, and activation; the user configures the records at their registrar.

## Tools

| Tool | Purpose |
|------|---------|
| `connect_custom_domain` | Start connecting a domain to a site (`siteId` from `list_sites`, plus the domain). Creates a pending domain and returns the DNS records to configure. |
| `get_custom_domain_status` | Inspect a domain's activation state and re-read its required DNS records (`customDomainId`). |
| `verify_custom_domain` | Check DNS for a pending or stale domain and activate it once records have propagated (`customDomainId`). |

Keep the `customDomainId` returned by `connect_custom_domain` — the other two tools take it, and there is no tool that lists a site's domains.

## Full Domain Setup Flow

1. Ask the user which domain they want to connect (e.g. `shop.example.com` or `example.com`).
2. Call `list_sites` to determine which site to attach the domain to:
   - If the project has one site, use that site's `id`.
   - If the project has multiple sites, ask the user which site the domain should point to.
3. Call `connect_custom_domain` with the `siteId` and the domain. Keep the returned `customDomainId`.
4. Present the returned DNS records clearly (see DNS Instructions below).
5. **If the user connected a root domain** (e.g. `example.com`), tell them they should also consider connecting `www.example.com` separately. Many registrars automatically redirect the root domain to the `www` version — if that redirect is in place and `www` isn't connected in Replo, visitors get a "refused to connect" error. Do not connect it without their explicit permission — just let them know.
6. Tell the user to configure DNS at their registrar.
7. Once the user confirms DNS is configured, call `verify_custom_domain` with the `customDomainId`.
8. If verification succeeds, confirm the domain is active. If DNS has not propagated, the tool returns the domain's current status rather than failing — tell the user to wait and try again.

## DNS Instructions

Both `connect_custom_domain` and `get_custom_domain_status` return `dnsInstructions` with a discriminated `type` field ("root" or "subdomain") and `records`. Present the records in a table.

**Subdomains** (e.g. `shop.example.com`) require one CNAME record pointing to `{reploSubdomain}.reploshops.com`.

**Root/apex domains** (e.g. `example.com`) require multiple records — the user must add ALL of them:
- Two A records — Cloudflare anycast IPv4 addresses (both are needed for redundancy)
- Two AAAA records — Cloudflare anycast IPv6 addresses (both are needed for redundancy)
- One TXT record — `replo-verify={token}` at `_replo-verification.{domain}` (proves ownership, expires in 7 days)

The exact values are in the tool response — relay them directly, don't hardcode IPs or tokens. Present each record as its own row in a table so the user adds all of them.

## Retrieving DNS Instructions for an Existing Domain

If a domain is already pending and the user needs to see the DNS instructions again, call `get_custom_domain_status` with the `customDomainId`. This returns the full instructions including the verification token for root domains. Do NOT tell the user to remove and reconnect the domain just to see its DNS records.

## Verifying and Activating a Domain

Call `verify_custom_domain` with the `customDomainId`. Replo then:

1. Checks the DNS records are correctly configured (CNAME for subdomains, A/AAAA + TXT for root domains).
2. Registers the domain as a custom hostname in Cloudflare for SSL provisioning.
3. Marks the domain active and supersedes any stale siblings.
4. Updates routing so traffic reaches the site.

If DNS hasn't propagated yet, the tool reports the domain still pending instead of failing. Tell the user to wait — propagation usually takes a few minutes but can take up to 48 hours — then call it again.

## Checking Status

Call `get_custom_domain_status` with the `customDomainId` and explain the state:

| State | Meaning |
|-------|---------|
| `isPending: true` | Connected but not yet verified — DNS may not be configured yet |
| `isPending: false`, `isStale: false` | Active and serving traffic |
| `isStale: true` | Previously active, now superseded — can be reclaimed by re-verifying |

## Removing a Domain

There is no public tool for disconnecting a domain. If the user wants one removed, ask a Replo session to do it, or point them at the Replo dashboard.

## Troubleshooting

**"Refused to connect" or site not loading after setup:**

If the user set up a root domain (e.g. `example.com`) and the site isn't loading, check whether their browser or registrar is redirecting to `www.example.com`. Many registrars add a default root → www redirect. If that redirect exists and `www.example.com` isn't connected in Replo, visitors land on `www.example.com`, which has no custom hostname or routing entry — producing a "refused to connect" error or a 404.

To diagnose: the root domain will return a 301 redirect to the www version, and the www version will fail. The fix is to also connect `www.example.com` as a subdomain (it needs its own CNAME record). The root → www redirect itself is controlled by the user's registrar, not by Replo.

## Error Handling

| Error message | What to tell the user |
|---------------|----------------------|
| "domain is already in use by another workspace" | This domain belongs to a different Replo account. Contact support or use a different domain. |
| "domain is already used by another project" | This domain is on another project in your workspace. Remove it there first. |
| "domain is already active" | Already verified and working — no action needed. |
| "Site not found" | The siteId doesn't exist or doesn't belong to this project. Call `list_sites` to get valid site IDs. |
| "DNS records not yet configured" | DNS hasn't propagated yet. Wait a few minutes and call `verify_custom_domain` again. |
| "missing a verification token" | The root domain's verification token expired. Ask a Replo session to reset the domain so a new token is issued. |
