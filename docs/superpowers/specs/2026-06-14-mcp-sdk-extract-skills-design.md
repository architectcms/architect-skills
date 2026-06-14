# Design: MCP awareness, SDK setup, content extraction & MCP install skills

Date: 2026-06-14
Status: Approved (brainstorming) — pending implementation plan

## Problem

When the architect skills were moved out of `../architect` into this repo, they were
rewritten to drive the `architect` CLI exclusively and lost any awareness of the MCP
server. We want to:

1. Keep the CLI as the **preferred** path, but let skills also work when the user has the
   **Architect MCP server** installed.
2. Make the MCP server installable from this plugin (today it is unpublished and lives only
   in `../architect/services/mcp`).
3. Add app-side skills for the **SDK** (delivery + preview clients) including an example `.env`.
4. Add a skill that analyzes a project for hard-coded strings/assets, models them, and uploads
   them to the CMS.
5. Make `architect-setup` the clear entry point and document **where each kind of API key goes**.

## Findings (current state)

- **10 skills, all CLI-only.** `architect-setup` installs `@architectcms/cli` (v0.2.0) globally
  and logs in with a **management** key (`arch_mgmt_…`) stored at `~/.architect/credentials.json`.
- **SDK** `@architectcms/sdk` (v0.3.0) is one npm package exporting three clients:
  - `ArchitectDelivery` — delivery key `arch_delivery_…` (published content)
  - `ArchitectPreview` — preview key `arch_preview_…` (drafts/unpublished)
  - `ArchitectManagement` — management key `arch_mgmt_…`
  It is a **per-project dependency**, not a global install.
- **MCP server** `architect/services/mcp` → `@architect-cms/mcp` v1.0.0. Transport is
  **`StdioServerTransport` only** — a local Node subprocess that proxies to the Architect REST API
  via `ARCHITECT_URL` + `ARCHITECT_API_KEY` + `ARCHITECT_ORG_ID` + `ARCHITECT_ENV_ID`. There is **no
  remote/HTTP variant** anywhere in the repo, it is **not on npm**, and it is **not deployed**. It
  dynamically generates per-model CRUD tools (`list_{model}`, `get_{model}`, `create_{model}`,
  `update_{model}`, `delete_{model}`) plus static tools (`list_models`, `search_entries`,
  `get_entry_references`, `list_environments`, `switch_environment`).

## Key decisions

| # | Decision |
|---|----------|
| 1 | **MCP = local stdio, bundled in the plugin.** The `architect-cms` plugin ships a `.mcp.json` running `npx -y @architectcms/mcp`. The skill documents the remote `type:http` config as a drop-in for a future hosted endpoint. **Prerequisite (out of repo):** publish the server to npm as `@architectcms/mcp` (consistent scope) from the `architect-sdk` monorepo. |
| 2 | **One `architect-sdk` skill** covering install + delivery and/or preview client scaffold + example `.env`. No separate delivery/preview skills (same package). |
| 3 | **MCP-awareness via a shared reference.** A single `references/mcp-equivalents.md` maps CLI commands → MCP tools. Each of the 10 existing skills gets a **one-line pointer** to it. |
| 4 | **`architect-extract` is propose → confirm → apply.** Scan, write proposed model + entry JSON locally for review, push only after approval. Never auto-mutates the CMS. |
| 5 | **No separate "global installer" skill.** CLI global install stays in `architect-setup`; SDK per-project install lives in `architect-sdk`. |

## Why local over remote MCP (rationale)

Local stdio is the better default today: the management key stays on the user's machine, it works
against cloud / self-hosted / `localhost` backends via `ARCHITECT_URL`, and it is shippable now
(only needs publishing). It still delivers the "install is just config in the plugin" experience
(`npx` config, no clone). Remote hosted MCP is a nicer end-state for cloud-only SaaS users and
non-process MCP clients, but it requires Architect to build/operate/secure a multi-tenant service
with proper auth — backend work that does not exist yet. Remote is designed as a future drop-in,
not a blocker.

## Skills — in scope

### New: `architect-mcp`
Install/configure the Architect MCP server.
- **Primary (local):** write/merge a project `.mcp.json` entry that runs `npx -y @architectcms/mcp`
  with `ARCHITECT_URL`, `ARCHITECT_API_KEY` (management), `ARCHITECT_ORG_ID`, `ARCHITECT_ENV_ID`.
- Cover Claude Code (`.mcp.json`), Claude Desktop (`claude_desktop_config.json`), and a generic
  client note.
- **Remote drop-in:** documented `type:http` block for `https://mcp.architectcms.com` (when it exists).
- Explain the relationship to the CLI (CLI preferred for scripting/CI; MCP for in-agent CRUD) and
  link to `references/mcp-equivalents.md`.
- Security: never commit `.mcp.json` containing a real key; use env var interpolation / placeholders.

### New: `architect-sdk`
Set up `@architectcms/sdk` in the user's app.
- `npm install @architectcms/sdk` (per-project).
- Scaffold the client(s) the user wants:
  - **Delivery** — `ArchitectDelivery`, key `arch_delivery_…`.
  - **Preview** — `ArchitectPreview`, key `arch_preview_…`.
- Write an example **`.env.example`** (and `.gitignore` `.env`) with the right prefixes:
  ```
  ARCHITECT_ORG_ID=org_…
  ARCHITECT_ENV_ID=env_…
  ARCHITECT_DELIVERY_KEY=arch_delivery_…
  ARCHITECT_PREVIEW_KEY=arch_preview_…
  ```
- Minimal usage snippet (fetch entries, image transforms, `withContext`) pulled from the SDK README.
- Note management-client usage exists but app code should prefer delivery/preview keys.

### New: `architect-extract`
Analyze a project for hard-coded content and migrate it into the CMS. **propose → confirm → apply.**
1. **Scan** source for hard-coded UI strings and asset references (images, etc.).
2. **Propose models** inferred from the patterns; write definitions to local JSON.
3. **Propose entries**; write entry JSON locally.
4. **User reviews/edits** the proposed files.
5. **Apply** only on approval: push models (via `architect-models` flow), then entries
   (via `architect-entries` flow); optionally upload assets.
- Defers to `architect-models` / `architect-entries` for the actual push mechanics rather than
  duplicating them. MCP-aware via the shared reference.

### Changed: `architect-setup`
- Remains the entry point. Add pointers to `architect-sdk` and `architect-mcp`.
- Add a short **"Where your API key goes"** table distinguishing the three locations:
  | Use | Key type | Location |
  |-----|----------|----------|
  | CLI | `arch_mgmt_…` | `~/.architect/credentials.json` via `architect login` |
  | MCP server | `arch_mgmt_…` | `.mcp.json` env (`ARCHITECT_API_KEY`) |
  | SDK (delivery/preview) | `arch_delivery_…` / `arch_preview_…` | project `.env` |

### Changed: existing 10 skills (MCP retrofit)
- Add `references/mcp-equivalents.md` (CLI command → MCP tool mapping) at the plugin root references.
- Add a one-line pointer to each skill, e.g.:
  > *Prefer the CLI. If the Architect MCP server is installed (`architect-mcp`), the equivalent
  > tools are in `references/mcp-equivalents.md`.*

## Out of scope / dependencies

- **Publishing `@architectcms/mcp` to npm** — required for decision #1; done in the `architect-sdk`
  monorepo, not this repo. Until then the skill can fall back to documenting the local path.
- **Hosted/remote MCP endpoint** — future backend work; only documented as a drop-in here.
- No changes to the CLI or SDK source themselves.

## Versioning

Feature release → bump `marketplace.json` and `plugin.json` to **0.3.0** in sync; mirror in the
Codex plugin manifest. Conventional Commits per skill.

## Open questions

- Exact npm name once published — assume `@architectcms/mcp`; confirm at publish time.
- `architect-extract` asset upload: confirm the CLI supports asset upload, or scope extract to
  strings + asset *references* first and add asset upload as a follow-up.
