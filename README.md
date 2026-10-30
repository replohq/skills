# Replo Agent Skills

Replo's skills live in the Replo MCP connector's `skills` tool, so they are current on every Replo deploy: how to build pages backed by real Shopify data, publish, connect custom domains, apply branding, query analytics, and use each integration.

This repository ships one skill, `replo`, which tells your agent to call `skills` to get the full list of skills, then `skills` with a name to load the one it needs, before doing Replo work. Connect the MCP server below so the agent can call it.

## Install the skill

```bash
npx skills add replohq/skills
```

Works with whichever agent you use: Claude Code, Cursor, Codex, OpenClaw, opencode, and others. Update later with `npx skills update`.

## Install the plugin

The shared Claude Code, Cursor, and OpenClaw bundle includes the `replo` skill and the Replo MCP connection. `npx skills add` installs only the skill; use the plugin steps below to load both. Choose the plugin or `npx skills add`, not both, or the skill loads twice.

**Claude Code:** install it in one step, then restart Claude Code:

```bash
claude plugin install replo --marketplace replohq/skills
```

If your Claude Code version doesn't recognize `--marketplace`, run `claude plugin marketplace add replohq/skills` and then `claude plugin install replo@replo`.

Run `/mcp` to confirm the Replo server is connected. Update with `claude plugin marketplace update replo && claude plugin update replo@replo`.

**Cursor:** clone it into Cursor's [local plugins folder](https://cursor.com/docs/plugins#test-plugins-locally):

```bash
mkdir -p ~/.cursor/plugins/local
git clone https://github.com/replohq/skills.git ~/.cursor/plugins/local/replo
```

Run **Developer: Reload Window**, then open **Customize** to confirm the Replo skill and MCP server appear. On Cursor Enterprise, an admin must turn on **Allow Local Plugin Imports** first. Update with `git -C ~/.cursor/plugins/local/replo pull --ff-only` and reload again.

**OpenClaw:** install it from [ClawHub](https://clawhub.ai/replohq), then restart the Gateway:

```bash
openclaw plugins install clawhub:@replohq/replo
```

Run `openclaw plugins inspect replo` to confirm the skill and the Replo MCP server are listed. Update with `openclaw plugins update replo`.

## Connect the MCP server

The MCP server serves Replo's skills and lets your agent act on your account. Connect to the hosted **Streamable HTTP** server at:

```text
https://api.replo.app/v1/mcp
```

Reading skills and competitor research work before sign-in. To work with your projects, sites, product data, assets, and analytics, complete the browser OAuth sign-in. A Replo API key does not replace this sign-in. See [connecting other AI clients](https://docs.replo.app/mcp/connect-with-other-ai-clients) for per-client setup.

Try: “Review the homepage of [site name] in [project name] and suggest three improvements. Make no changes yet.”

[server.json](server.json) contains the MCP Registry metadata for this hosted service. This repository distributes client setup and the `replo` skill; the server runs on Replo's infrastructure.

## What's here

`skills/replo/SKILL.md` is the skill. It stays short and fixed: the skills it points to update with Replo, not with this repository. The same file is uploaded with the Replo listings on claude.ai and ChatGPT; claude.ai accepts only six frontmatter keys (name, description, license, compatibility, metadata, allowed-tools) and truncates the listing at 1,536 characters.

## Contributing

These files are generated from Replo's source tree on every merge, so pull requests here are closed with a pointer. File an issue instead, or contact [support@replo.app](mailto:support@replo.app) — fixes land in our monorepo and reach this repo on the next sync.

## License

Skills are MIT-0 (MIT No Attribution); see [LICENSE.skills](LICENSE.skills). The plugin itself remains proprietary; see [LICENSE](LICENSE). The hosted MCP service is licensed separately under Replo’s terms.
