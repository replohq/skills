---
name: publish
title: Publish a Site
summary: Take a site live, and diagnose a failed build or deploy.
description: 'REQUIRED when the user asks to publish, deploy, go live, push live, make changes live, or ship. You MUST load this skill before taking any publish action. The `publish_site` tool is the ONLY way to publish — never run build or deploy commands directly via bash. Triggers: "publish", "deploy", "go live", "make it live", "push live", "ship it", "publish the site", "deploy my changes", "publish all", "publish everything", "publish all my sites".'
tools: publish_site, list_sites
---

# Publish

**You MUST use the `publish_site` tool to publish. Do NOT run build or deploy commands manually.** The publish tool handles the entire pipeline: production build, packaging, upload, and deployment. There is no other way to publish.

## One Site Per Publish Request (pipeline constraint)

**CRITICAL: You may only publish ONE site per publish request.** Even if the user asks to "publish everything", "publish all my sites", or "deploy all sites", you MUST publish them **one at a time, sequentially**. Never call the `publish_site` tool for multiple sites in parallel or in a batch. The publish pipeline does not support concurrent publishes — running multiple publishes simultaneously will cause build corruption, race conditions, and failed deployments.

This is a constraint on the publish pipeline only. It does not limit how much
you build in one turn or one session: creating several pages on a site before
publishing once is normal.

If the user asks to publish multiple sites:

1. Identify all the sites to publish via `list_sites`.
2. Confirm with the user which sites they want published.
3. Publish the first site. Wait for it to fully succeed (build + deploy complete).
4. Only after the first publish succeeds, publish the next site.
5. Repeat until all requested sites are published.

If a publish fails mid-sequence, stop and report the failure. Do not continue publishing remaining sites until the failure is resolved.

## User Must Explicitly Request Publishing

**CRITICAL: Never build or publish unless the user has explicitly asked you to.** Making code changes does NOT imply a request to publish. Even if the user says "make this change and let me see it", that does **not** mean publish — the dev server preview is sufficient. You must only begin the publish workflow when the user uses clear language like "publish", "deploy", "push it live", "make it live", or otherwise unambiguously asks for their site to be published.

If you are unsure whether the user wants to publish, **ask them** rather than proceeding.

## Do Not Move Project Files

**CRITICAL: Never move or relocate project files out of their original directory structure.** The build expects the Next.js app at the repo root — `app/`, `components/`, `package.json`, and the config files where the site scaffold put them. If you have moved any files or directories, move them back before publishing, or the build will fail.

## Determine Which Site to Publish

Before calling `publish_site`, call `list_sites` to get site IDs.

- If the prompt provides a specific `siteId`, that IS the site to publish. Do not ask — the user already chose this site.
- Otherwise, if the user named a site (e.g. "publish the blog"), match by site name.
- Otherwise, apply the default-site rule from your system prompt, which ends in
  asking the user when nothing resolves. Never guess — publishing the wrong
  site overwrites its live deployment.

Pass the `siteId` to the `publish_site` tool. That is the whole input — the tool finds the site's folder on disk itself, so never go looking for it.

## Commit Before Publish

Before starting a build, check for uncommitted changes with `git status --porcelain`.

If there are untracked or modified files, commit and push them before publishing. The `publish_site` tool tags the HEAD commit at publish time, so publishing from a dirty working tree would mean the tag points at a commit that does not represent what was actually deployed.

If the working tree is already clean, do not create an empty commit.

## Build/Publish Order

First, call the `publish_site` tool immediately.

If publish fails due to **runtime** issues, reproduce them against a local dev server (`pnpm dev`) and fix them there. For **build** failures (TypeScript, module resolution), follow "Handling Build Failures" below — a running dev server cannot see those.

The `publish_site` tool runs the production OpenNext build for the specified site directory automatically before packaging and deployment.

## Handling Build Failures

The build fails fast: `next build` stops at the **first** type error, so the error output is never a complete list of what is broken. Treat every reported error as one instance of a pattern until a full search proves otherwise.

If the publish tool fails during build/validation:

1. Read the error output carefully — it names the first failing file, not all of them.
2. Before fixing anything, search the whole site for the same pattern. For a module-resolution or bad-import error, sweep every source directory (`app/`, `components/`, `lib/`, …) for the failing specifier with **no source exclusions** — never exclude the file or directory the error came from; sibling files often share the same broken import. Only dependency and build output are excluded, because their matches are benign:

   ```bash
   grep -rln --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=.open-next "the/failing/import-path" .
   ```

   Fix every file the sweep finds, then re-run the sweep and confirm zero matches remain.
3. Verify with the same check that failed, not a different one. Type errors are invisible to the dev server — pages render fine while `next build` still fails, so a clean-looking dev server is NOT evidence the build will pass. Before retrying publish, run the type check directly in the site directory and require it clean:

   ```bash
   pnpm exec tsc --noEmit
   ```

   This takes seconds; a publish retry takes minutes and may require the user to approve again. Never retry publish with a known-dirty type check.
4. Re-run publish (which reruns the build step).
5. Repeat until the build succeeds.

A running dev server remains the right way to chase **runtime** errors — pages crashing, data failing to load. It is the wrong verification for build/type errors.

**Important:** When fixing build errors, make only the minimal changes needed to resolve each specific error. Do NOT delete, comment out, or stub out existing functionality to get the build to pass. If a build error cannot be resolved with a small, targeted fix, report the problem to the user and ask for guidance rather than removing code.

Do not treat publish as successful until the build step and deploy step both complete.

## Build-Phase Problems vs. Infrastructure Problems

There are two fundamentally different classes of failure, and you should treat them differently.

**Build-phase problems in the Next.js codebase — fix these.** These originate in the project's source code and are within your control: TypeScript errors, ESLint failures, module resolution / missing import errors, invalid JSX, broken page exports, or other compile-time issues in the app code. Read the error, make a minimal targeted fix in the source files, and retry publish.

**Infrastructure problems — do NOT try to fix these.** These come from the platform, not the project's code, and are outside your control. Examples include the publisher returning 500s, build processes running out of memory (OOM kills), upload/deploy/network timeouts, R2 or deployment-service errors, and other transient platform failures. Do NOT attempt to diagnose or work around the underlying infrastructure — do not edit config to reduce memory usage, do not patch around the publisher, do not try to investigate the platform internals. Instead, simply **retry the publish a few times**. If it continues to fail after a few retries, **give up and report the failure to the user**, briefly explaining that it appears to be a platform/infrastructure issue rather than a problem with their site.

## Standalone Output Problems

Publish deploys the `output: "standalone"` tree Next writes to `.next/standalone`. Three `next.config.ts` keys have to be right for that tree to exist where publish looks for it, and **all three are fixable code issues, NOT infrastructure problems — do not just retry.** The failure names the key you got wrong:

- **`distDir`** — the build landed somewhere other than `.next` (usually `.next-dev`, from a config that dropped the production branch). **Fix:** `distDir: isBuild ? ".next" : ".next-dev"`.
- **`output: "standalone"`** — missing, so the build emits nothing deployable. **Fix:** add it.
- **`outputFileTracingRoot: "/"`** — set to something else (often a monorepo root such as `join(here, "..", "..")`), which changes how deeply the tree nests. **Fix:** set it to `"/"` for production builds.

Losing these is what happens when `next.config.ts` gets rewritten by hand. If the site can import `siteConfig()` from `@replohq/sdk/next/config`, spread it (`export default siteConfig()`) rather than hand-rolling the object — it pins all three plus `turbopack.root` together. Otherwise add the one missing key; do not restructure a working config.

Make the fix, commit it, then retry publish. A raw `cp: cannot stat '.../.next/standalone/...'` with no explanation is one of these same three causes — check them in the order above, since a build that never produced the tree cannot be fixed by tuning where the tree nests.

## Publishing

Call the `publish_site` tool to build and deploy. It handles running the OpenNext build, validating artifacts, compressing upload payloads, and triggering deployment. It returns the published URL on success.

### Handling publish errors

The `publish_site` tool validates artifacts before uploading. Handle its errors as follows:

- **Missing `open-next.config.ts`**: Create it at the site root with exactly `export { default } from "@replohq/sdk/next/open-next";`, commit, and call `publish_site` again.

- **Missing `<ReploProvider>` in `app/layout.tsx`**: The site's root layout must wrap the app in `<ReploProvider>` (from `@replohq/sdk/providers/replo-provider`). It mounts the Replo runtime layer — cart (`useCart`, add-to-cart, buy-now), analytics and the first-party pixel, consent-gated tracking scripts, and the React Query client the SDK data loaders depend on. Without it the site still builds and renders, but those hooks throw at runtime and no analytics fire, so publish is blocked before the build. This is a site-level problem, never infrastructure. Add the import and wrap the app's children in `<ReploProvider>` inside `<body>` (keeping any existing `<SandboxRuntime>`, `<ReploScripts />`, and `{children}`), commit, and call `publish_site` again.

- **Dependency preflight failures** (no `node_modules`, `@opennextjs/cloudflare` missing or outside the supported version window, `react`/`react-dom` version mismatch): The publish tool checks the site's installed dependencies before building. These are site-level problems, never infrastructure — diagnose from the error message, fix the site's dependencies, and call `publish_site` again.

- **Missing or invalid artifacts** (e.g. `.open-next/` missing, `worker.js` empty, `assets/` missing, `wrangler.jsonc` missing): The build output is broken or incomplete. Fix the underlying app issue and call `publish_site` again (it will rebuild).

- **Stale artifacts** (`.open-next/worker.js` is older than the latest commit): Something has changed since the build was produced. Tell the user:

  > The build artifacts are out of date — there have been changes since the last build. This could mean someone else pushed changes while you were working. Would you like me to rebuild and try publishing again?

  Wait for the user to confirm before rebuilding and re-publishing.

### Typical flow

1. Make code changes.
2. Call `list_sites` to get the target site's `siteId`, unless the prompt already provided one. Resolve the site per "Determine Which Site to Publish" above.
3. Run `git status --porcelain`. If there are uncommitted changes, commit and push them. If the working tree is already clean, do not create an empty commit.
4. Call the `publish_site` tool with `siteId` (it runs the build).
5. If the build step fails due to a code-level issue (TypeScript errors, etc.), sweep for every occurrence of the failing pattern, fix them all, verify with `pnpm exec tsc --noEmit`, and retry publish.
6. If publish fails due to missing/invalid artifacts, rebuild and retry.
7. If publish fails due to an infrastructure problem (publisher 500s, out-of-memory builds, deploy timeouts), do not try to fix the infrastructure — retry a few times, and give up and report to the user if it keeps failing.
8. Share the published URL(s) with the user. If the user has custom domains configured, mention only those (not the replosites.com domain). If there are multiple custom domains, list them all. Only mention the replosites.com domain if no custom domains are configured.
