# Replo Agent Skills

Skills that teach any AI coding agent how to build on [Replo](https://replo.app) — pages backed by real Shopify data, publishing, custom domains, branding, analytics, and the integrations.

## Install

```bash
npx skills add replohq/skills
```

Installs into whichever agent you use — Claude Code, Cursor, Codex, OpenClaw, opencode, and others. To pick a subset:

```bash
npx skills add replohq/skills --list
npx skills add replohq/skills --skill building-replo-pages
```

Update later with `npx skills update`.

## Connect the MCP server

Skills tell an agent how to build; the MCP server lets it act on your account. Connect it at `https://api.replo.app/v1/mcp` — see [connecting other AI clients](https://docs.replo.app/mcp/connect-with-other-ai-clients) for per-client setup.

## What's here

`building-replo-pages` is the one to read first: a Replo site is a Next.js App Router repo, and pages get their catalog data from Shopify data loaders rather than hardcoded values. The rest cover publishing, custom domains, branding, analytics queries, Shopify, and the Klaviyo, Okendo, Rebuy, Smile.io, and Statsig integrations.

## Contributing

These files are generated from Replo's source tree on every merge, so pull requests here are closed with a pointer. File an issue instead, or contact Replo support — fixes land in our monorepo and reach this repo on the next sync.

## License

MIT
