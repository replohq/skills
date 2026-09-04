---
name: replo
description: Use whenever a request touches Replo through the Replo connector: ecommerce sites and pages, brand kits (Brand Studio), Insights analytics and reports, Replo-managed products and orders, media assets, integrations, custom domains, scheduled tasks, skills, or plans. Explains how a Replo project is structured, which connector tools read or write each part, and what can only be done by describing it to a Replo session. Triggers include "build me a landing page", "set up my brand", "how is my site performing", "publish my site", "connect my domain", "schedule a weekly report", "make me a store".
license: SEE LICENSE IN LICENSE
compatibility: Requires the Replo connector and a Replo account.
---

# Replo

Replo builds and runs ecommerce storefronts with an AI agent. This server lets you operate a user's Replo account. Call list_projects first and use returned IDs instead of guessing. Make sure you are working in the project the user requests. Double check with them if you are not sure.

## How Replo is organized
- **Workspace → Project → Site → Pages.** A project is one store or brand. A project can hold several **sites**; each site is its own deployable web app published to `{subdomain}.replosites.com` or a connected custom domain. At most one site per project is the **default** site.
- **Pages** live inside a site (routes like `/`, `/about`, `/products/[handle]`). A site's `name` is a dashboard label, not the SEO title.
- **Default-site rule:** when the user does not name a site, use the project's default site; if the project has no default, ask which site to use — never assume.
- **Publishing:** work is draft and private until published. Publish only when the user explicitly asks; making changes never implies a request to publish. Publish one site per request. After a publish, further edits are draft again until the next publish.

## What lives in a Replo project
A project is one brand or store. Everything below belongs to a project, and every tool here takes a projectId (or an ID you got from one). Replo presents the project as a set of apps; the agent inside a session can use all of them. You can read or write some of them directly with tools; the rest you reach only by describing what you want in a session prompt.

### Sites and pages — Site Builder
A site is a deployable storefront or web app; a project can hold several, and one may be the default. Pages live in a site as routes (`/`, `/about`, `/products/[handle]`). Building is conversational: the agent composes pages from a library of templates and sections and adapts them to the brand and products in the project. It works autonomously on drafts; nothing is public until published.
- Tools: list_sites, update_site (dashboard display name only), publish_site.
- Prompt-only: creating, editing, restyling, or removing pages and sections; SEO titles and metadata; navigation; forms; anything inside a page. Describe pages by route and purpose.

### Brand Studio
Where the brand lives: colors, fonts, logos, imagery, and a business profile (what the store sells and for whom). Set up once, used everywhere — sites, emails, product copy. The agent can generate it from an existing website URL, walk the user through it, or build it by hand. Applying the brand to a site is a design-token change on that site, not a rebuild.
- Tools: none.
- Prompt-only: create or update the brand kit or business profile, apply a brand to a site, restyle a site to match a brand, and report the brand's colors, fonts, and logo URLs back to you. Look brand values up this way rather than asking the user to retype what Replo already holds.

### Products and checkout
A project may hold two catalogs at once: Replo-managed products (sold through Replo checkout) and products synced from Shopify or another connected store (sold through that store's checkout). Search both before telling a user a product is missing.
- Tools: find_products, get_product, create_product, update_product — Replo-managed products only.
- Prompt-only: product pages and collection pages, bulk catalog edits, product copy and imagery, working with Shopify-synced products, checkout and cart behavior.

### Orders
Orders placed through Replo checkout. Amounts are integers in the currency's minor units (2500 = $25.00 USD) paired with a currencyCode.
- Tools: find_orders, get_order.
- Prompt-only: fulfillment workflows, refunds, customer follow-up, order reporting.

### Assets versus Files
Assets is the project's media library — images, video, fonts, documents that pages and emails reference. Files is a separate private drive of documents the user and the agent work from (briefs, spreadsheets, drafts), with folders and sharing. upload_asset and find_assets touch Assets only.
- Tools: find_assets, upload_asset.
- Prompt-only: generating or editing images, organizing the library, anything in Files.

### Insights
Traffic, engagement, and sales dashboards for the project's sites, built from Replo's own event data. Includes report templates and custom reports the agent can build from a description.
- Tools: query_replo_analytics — read-only SQL over the project's analytics tables (events_computed, daily_page_rollups, daily_namespace_rollups, daily_namespace_purchase_rollups, currency_exchange_rates). Queries are scoped to the project automatically.
- Prompt-only: building or editing a saved report or dashboard, recurring performance summaries, analysis that needs the agent to interpret and act.

### Integrations
Connections to the tools a store runs on — email platforms, analytics, ads, Shopify, and more — so the agent can read from and act on them. A connection can be project-wide or personal to one user. Connecting a tool does not install its tracking on the site; that is a separate, explicit request.
- Tools: none.
- Prompt-only: listing what is connected, using a connected tool (send a campaign, pull ad spend, sync products), adding tracking scripts to a site. Connecting a new integration requires the user to authorize it in the Replo dashboard.

### Custom domains
A site publishes to `{subdomain}.replosites.com` by default. Connecting a custom domain returns DNS records the user must add at their registrar; verification activates the domain once they resolve. A subdomain (shop.example.com) needs one CNAME; an apex (example.com) needs several records — suggest connecting www alongside an apex.
- Tools: connect_custom_domain, get_custom_domain_status, verify_custom_domain.
- Prompt-only: nothing; domains are fully tool-managed.

### Tasks
A task is a prompt with a schedule attached. Replo runs the prompt in its own session at each occurrence — weekly reports, daily inventory checks, recurring content updates.
- Tools: list_tasks, get_task, create_task, update_task, delete_task.
- Prompt-only: nothing; tasks are fully tool-managed. A task's prompt can do anything a session prompt can.

### Skills and Plans
Skills are reusable playbooks (instructions plus reference files) the agent loads automatically when a request matches; users install them from a shared library or write their own. Plans are written checklists the agent proposes before larger work; the user reviews, then the agent builds. You rarely need to name either — ask for the outcome and the agent picks the skill or proposes a plan.
- Tools: none.
- Prompt-only: install, create, or edit a skill; ask for a plan before building; approve or revise a plan.

### CMS, Settings, and memory
CMS holds structured content (blog posts, collections of records) that pages render. Settings covers project configuration such as team, billing, and site-level options. Memory is what the agent remembers: user memory is private and follows the user across projects; project memory is shared by everyone on the project. Brand identity is not memory — it lives in Brand Studio.
- Tools: none.
- Prompt-only: all of it. Tell the session what to remember, what content to create, or what setting to change; it will ask the user when a change needs confirmation.

### Sessions are where the work happens
Everything marked prompt-only above is done by starting a session (start_agent_session) or continuing one (send_agent_message) with a plain-language description of the outcome. The agent has every app above available; you do not need to name the app, only the result.

## Tools read and write records; sessions do the work
The tools here manage records: projects, sites, Replo-managed products, orders, media assets, analytics queries, custom domains, and scheduled tasks. Everything that *builds* — pages and their content, brand kits (Brand Studio), saved Insights reports, integrations, emails, CMS content, skills, plans, settings, and what the agent remembers — has no tool. It is done by describing the outcome in a session prompt. If you need something and no tool matches, that is the signal to start or continue a session, not to tell the user it is unsupported.

## Sessions
A session (start_agent_session) is a persistent, project-scoped conversation with the Replo agent. The prompt is free text; the agent has the project's site code checked out and every Replo app available. Mechanics to plan around:
- A finished session stays open. send_agent_message continues it with its full context; a new session starts fresh (only project memory carries over).
- A session holds the site's repository while it works. Two sessions editing the same site at the same time write to the same repository and will conflict; sequence them or keep that work in one session.
- Sessions edit drafts. Nothing is live until publish_site is called for that site; do not ask the session to publish.
- Name the target site when the project has more than one (list_sites; use the default site when the user does not say). Refer to pages by route and say what each is for.
- The agent may stop to ask the user something (pendingInteractions); see below.

## Async loop
- start_agent_session and send_agent_message return before the turn completes. Poll get_agent_session every few seconds, backing off, while status is starting or running.
- A 404 immediately after starting a session means the sandbox is still cold-starting. Retry shortly rather than treating the session as invalid.
- When pendingInteractions is non-empty, stop polling, show the interaction to the user verbatim, and wait for their answer before calling resolve_agent_interaction. Never infer the answer yourself. send_agent_message returns 409 while an interaction is pending.
- Give the user the returned dashboardUrl so they can watch the work or take over.

## Teaching Replo how the user works
A skill is a standing instruction the Replo agent loads on every future session in that project, and it is how Replo adapts to a user instead of restarting from zero each time. When the user states a durable preference or a workflow they repeat — brand voice rules, how they want pages structured, steps they always take before publishing — offer to save it, and once they agree ask for it in a session: name it, say when it applies, and say what it instructs, one skill per idea rather than one catch-all. Do not save one-off requests, anything the user has not confirmed, or facts that belong in their brand kit.

## Local development
To set a project up on the user's machine: match the name with list_projects, following next_cursor until has_more is false; call list_sites for that project id and take the default site's clone_url; mint a key with create_api_key, passing repo.write only when the user will push. Then clone, install dependencies, and start the dev server. A local checkout and the Replo sandbox are two writers of the same repository, so pull before editing locally and push when done. Never force-push: Replo rebases onto the remote before saving, so force-pushing is the only way to discard the user's Replo-side work.

## Safety
Use read tools to resolve sites, products, assets, orders, and tasks before changing them. Confirm the project, the site, and the intended change before starting any session, scheduling any task, or making any write — session and scheduled-task prompts can modify live sites, and publish_site makes changes public.
