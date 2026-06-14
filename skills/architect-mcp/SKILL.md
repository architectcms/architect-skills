---
name: architect-mcp
description: Install and configure the Architect CMS MCP server so an agent can operate on content through MCP tools (list_/get_/create_/update_/delete_ per model, search_entries, switch_environment). Use when the user wants MCP-based content access instead of, or alongside, the architect CLI.
---

# architect-mcp

Set up the Architect CMS **MCP server** — a local stdio server (`@architectcms/mcp`) that proxies to the Architect API and exposes per-model CRUD tools to any MCP client.

## CLI vs MCP — which to use

- **CLI (`architect …`)** — preferred for scripting, CI, bulk import/export, and everything the other skills do.
- **MCP** — convenient for in-agent, conversational CRUD (`create_article`, `search_entries`, `switch_environment`). Optional.

The two are interchangeable for content operations; see `references/mcp-equivalents.md` for the command→tool mapping.

> Requires `@architectcms/mcp` on npm. Until it is published, use the local path fallback at the bottom.

## Credentials — shared with the CLI

The MCP server reuses the CLI's login. It resolves credentials in this order:

1. Environment variables (`ARCHITECT_API_KEY`, `ARCHITECT_ORG_ID`, `ARCHITECT_ENV_ID`, `ARCHITECT_URL`) — set by the plugin's config or a `.mcp.json`.
2. Otherwise, the CLI's `~/.architect/credentials.json` (written by `architect login`).

**So if you've already run `architect login` (see `architect-setup`), the MCP server works with no extra credentials** — you do not need to enter the key a second time. Only set the env vars when you want to override the logged-in account (e.g. a different environment for one project).

## Option A — Claude plugin (automatic)

If you installed the `architect-cms` Claude plugin, the MCP server is **already bundled**. When you enable the plugin, Claude Code offers fields for your Architect URL, **management** API key, organization id, and environment id — **all optional**: leave them blank to reuse your `architect login` credentials, or fill them in to override. The server starts automatically; verify with `/mcp` and look for the `architect-cms` server.

To reconfigure, re-run the plugin's configuration from `/plugin`.

## Option B — Manual (Claude Code project, Codex, other clients)

Add an entry to your project `.mcp.json` (never commit a real key — use an env var):

```json
{
  "mcpServers": {
    "architect-cms": {
      "command": "npx",
      "args": ["-y", "@architectcms/mcp"],
      "env": {
        "ARCHITECT_URL": "https://api.architectcms.com",
        "ARCHITECT_API_KEY": "${ARCHITECT_API_KEY}",
        "ARCHITECT_ORG_ID": "org_…",
        "ARCHITECT_ENV_ID": "env_…"
      }
    }
  }
}
```

For Claude Desktop, use the same block under `mcpServers` in
`~/Library/Application Support/Claude/claude_desktop_config.json`, but `npx` requires an absolute path or a shell wrapper there.

The key is a **management** key (`arch_mgmt_…`) — the same kind the CLI uses. See `architect-setup`. If you've already run `architect login`, you can **omit the `env` block entirely** — the server falls back to `~/.architect/credentials.json`.

## Option C — Remote (future drop-in)

When a hosted endpoint exists, replace the stdio block with an HTTP one:

```json
{
  "mcpServers": {
    "architect-cms": {
      "type": "http",
      "url": "https://mcp.architectcms.com",
      "headers": { "Authorization": "Bearer ${ARCHITECT_API_KEY}" }
    }
  }
}
```

## Verify

In Claude Code: `/mcp` lists `architect-cms` with tools like `list_models`, `search_entries`, `create_<model>`. Ask the agent to "list models" to confirm.

## Local-path fallback (before `@architectcms/mcp` is published)

Clone the source and point at it directly:

```json
{ "mcpServers": { "architect-cms": {
  "command": "node",
  "args": ["/absolute/path/to/architect-sdk/packages/mcp/index.js"],
  "env": { "ARCHITECT_URL": "…", "ARCHITECT_API_KEY": "…", "ARCHITECT_ORG_ID": "…", "ARCHITECT_ENV_ID": "…" }
}}}
```

## Related skills

- `architect-setup` — get a management API key and log in the CLI
- Every architect skill — see `references/mcp-equivalents.md` for the CLI↔MCP mapping
