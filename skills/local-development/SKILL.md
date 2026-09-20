---
name: local-development
title: Develop a Site Locally
summary: Clone a site, work on it locally, and push without publishing unfinished work.
description: "Use when cloning, pulling, pushing, or running a Replo site's code outside Replo — on a laptop, in CI, or from a coding agent — through git.replo.app or a site's clone_url. Also use when the user asks which branches Replo supports, whether Replo has pull requests or review, whether pushing to main deploys or publishes, why pushed changes did or did not go live, or how to keep unfinished local work from being published. Triggers: \"clone my site\", \"work on my site locally\", \"replo-git\", \"git.replo.app\", \"push to main\", \"does pushing deploy\", \"branches\", \"pull request\", \"review before publishing\", \"local dev server\"."
tools: list_projects, list_sites, create_api_key, publish_site
---

# Develop a Site Locally

Every Replo site is a Next.js repository at `https://git.replo.app/<siteId>.git`.
A local checkout is a second writer of that repository: the Replo editor and the
Replo agent commit to the same `main` branch you push to.

## How Replo treats your pushes

Read this before the first push. It explains how "I only pushed" can still end
with a change going live.

| Question | Answer |
|---|---|
| Which branch does Replo use? | Only `main`. Replo saves, syncs, previews, and publishes `main` and never reads any other branch. |
| Is there a pull request or review step? | No. Replo repositories have no pull requests, branch previews, or approvals. The review happens before you push to `main` (see below). |
| Does pushing to `main` publish? | No. A push only updates the repository. Nothing goes live until someone publishes. |
| Can a pushed commit go live later? | Yes. Replo brings `main` into its own copy of the site when someone opens the project in Replo, when that copy restarts, and whenever Replo saves its own changes. The next publish — the Publish button, a Replo agent the user asked to publish, or `publish_site` — builds that copy. Treat a push to `main` as "ready to publish". |
| Does `publish_site` publish my latest push? | Not necessarily. It publishes Replo's copy of the site, which may not have picked up a recent push, and it does not take a commit. Verify afterwards (see Publishing). |

## Set up

1. Resolve the site. If the user supplied a site ID, put the URL above in
   `REPLO_GIT_URL`; no MCP or dashboard access is needed. Otherwise, call
   `list_projects`, then `list_sites` for that project. Use the default site
   unless the user names one, and put its `clone_url` in `REPLO_GIT_URL`. People
   find the same URL in **Site Settings** > **General** > **Advanced** > **Git
   clone URL**.
2. Use an existing `REPLO_API_KEY` when the user supplied one with the required
   scope. Otherwise, mint a key with `create_api_key`: `repo.read`, plus
   `repo.write` only when the user will push. Write a new key straight to
   `REPLO_API_KEY`; never print it, commit it, or put it in a URL. Keys minted
   this way carry only repository scopes, so they cannot publish.
3. Point Git at the key for `git.replo.app` only, then verify and clone:

   ```bash
   git config --global credential.https://git.replo.app.helper \
     '!f(){ echo username=token; echo "password=$REPLO_API_KEY"; };f'
   git ls-remote "$REPLO_GIT_URL"
   git clone "$REPLO_GIT_URL"
   ```

   This helper also works in an isolated CI home. Do not interpolate the key
   into `git -c http.extraHeader=...`; the credential would be visible in Git's
   command-line arguments. Bearer and other options are in the [Replo Git
   docs](https://docs.replo.app/git/manual-setup).
4. Use versions pinned by the checkout. If it has no `packageManager`, `engines`,
   `.nvmrc`, or `.node-version`, use Node.js 22 and pnpm 10. Most sites then run
   locally with `pnpm install`, followed by `pnpm dev`.
5. Load the requested route and confirm it renders. A rendered route does not by
   itself prove that product data loaded successfully. When product data is
   expected, verify it against active products and confirm the expected data
   appears; an empty fixture or one with only inactive products leaves product
   loading unproven.

## Work without publishing by accident

- Start from the latest `main`: `git pull --ff-only`, or `git pull --rebase` when
  you already have local commits.
- Keep unfinished work on a local branch (`git switch -c my-change`). Merge it
  into `main` and push only once the user wants it included in the next publish.
- Routes with a `page.dev.tsx` draft and a `page.published.tsx` live copy go live
  when a publish copies the draft over the live copy. Edit `page.dev.tsx`, which
  is what `pnpm dev` shows. Never hand-edit `page.published.tsx`; publishing
  manages it. Publishing fails on any other `*.published.tsx` file under `app/`.
- Everything without a draft copy goes live on the next publish of the site,
  even a publish of a single page: layouts, shared components, styles, config,
  and dependencies. So does an edit to a plain `page.tsx` on a route that is
  already live. Keep unfinished changes to those files off `main`.
- Replo also commits to `main` (editor saves, agent work, automatic upgrades). If
  a push is rejected as non-fast-forward, run `git pull --rebase` and push again.
  Never force-push: it is the one way to erase work the user made in Replo.

## Before you push to main

Replo has no review gate, so this is the review:

1. `git fetch origin`, then `git log --oneline origin/main..HEAD` — only the
   commits you intend to share.
2. `git diff origin/main...HEAD` — read the whole change, especially files with no
   draft copy.
3. Preview in `pnpm dev` and run `pnpm exec tsc --noEmit`. Type errors in files
   the publish build typechecks (live copies, layouts, shared components, config)
   fail publishing even when the dev server renders. Drafts (`*.dev.tsx`) are
   excluded from the publish typecheck.
4. Check `git status --short`, stage only the intended paths, and inspect the
   staged diff. The dev server can generate root `AGENTS.md` and `CLAUDE.md`
   files; leave them untracked unless the user deliberately changed the site's
   shared instructions.
5. Push only when the user asks you to push. Asking for an edit is not a request
   to push, just as it is not a request to publish.

Teams that want pull-request review keep reviewed history in their own Git host
and push to Replo's `main` only after merging. Replo does not sync with other
Git hosts, so those teams must also merge Replo's own `main` commits back into
their repository before each push.

## Publishing

- Publish only when the user explicitly asks. Pushing, merging, or "let me see
  it" is not a request to publish; `pnpm dev` is the preview.
- Call `publish_site` with the `siteId`. Pass `promoteRoutes` to put only the
  named pages live; other pages keep their drafts, but files without a draft copy
  still go live.
- Do not work around a missing publish permission by asking for a broader key.
- After a recent push, have the user open the project in Replo before publishing
  so Replo's copy picks up `main`.
- Each publish tags its commit `published/<timestamp>/<publishId>`, and the tag
  can arrive shortly after the publish finishes. To confirm a commit shipped:

  ```bash
  git fetch --tags origin
  LATEST=$(git tag --list 'published/*' --sort=-refname | head -n 1)
  git merge-base --is-ancestor <your-commit> "$LATEST" && echo "published"
  ```

## Errors from git.replo.app

| Response | Meaning and fix |
|---|---|
| `Git authentication failed.` | No valid key was sent, or it was revoked. Ask for a valid key. |
| `Repository not found.` | Wrong URL or project, the key creator lost project access, or a `repo.read` key tried to push. Replo does not say which, so check all four. |
| `... rate limit exceeded. Retry after N seconds.` | Wait for the `Retry-After` interval before retrying. |

## Related Skills

- **building-replo-pages** — conventions for the code you edit locally.
- **publish** — the full publishing workflow and build-failure handling.
