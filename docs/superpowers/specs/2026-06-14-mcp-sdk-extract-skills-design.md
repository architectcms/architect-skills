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
3. Add an app-side skill for the **SDK** (delivery + preview clients) — install, example `.env`, and
   **fetching content from the CMS and wiring it into the app**.
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
| 1 | **MCP = local stdio, bundled in the plugin.** The `architect-cms` plugin ships a `.mcp.json` running `npx -y @architectcms/mcp`. The skill documents the remote `type:http` config as a drop-in for a future hosted endpoint. **This depends on publishing the server to public npm (see decision #6) — until then the skill is non-functional for external users.** |
| 6 | **Publish the MCP server to public npm.** Today it lives in the **private** `architect` repo (`services/mcp`, scope `@architect-cms/mcp`) and is unpublished, so no external user can install it. Move it to **`architect-sdk/packages/mcp`**, rename the scope to **`@architectcms/mcp`** (matching `cli`/`sdk`), wire it into the monorepo build/publish pipeline, and publish. Cross-repo prep (move + config) is in scope for this work; the actual `npm publish` is a manual step the maintainer runs. |
| 7 | **Add `architect assets upload` to the CLI.** The CLI has no asset command; only the SDK's management client can upload (`POST /api/v2/assets/upload`). Add a thin CLI command in `architect-sdk/packages/cli` wrapping that capability, so `architect-extract` (and future skills) can upload assets while staying CLI-driven. Published with the CLI; same cross-repo + manual-publish caveat as #6. |
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
Set up `@architectcms/sdk` in the user's app **and pull content into it**. This is the inverse of
`architect-extract`: extract pushes hard-coded content *up* to the CMS; this skill pulls it *down*
and wires it in.

**1. Install + configure**
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
- Note management-client usage exists but app code should prefer delivery/preview keys.

**2. Fetch content + hook it up** (depth: data layer + one wired example)
- Detect the app framework (Next.js, Remix, plain React/Vite, Node, etc.) and follow its conventions.
- Generate a **typed data-fetching layer** — small helper module(s) that instantiate the client from
  env and expose functions like `getArticles()` / `getArticle(id)`. Use `architect-types` so the
  helpers are type-safe against the schema.
- **Verify** the fetch works (list a model's entries) before editing app code.
- Wire **one representative spot** end-to-end as a copyable pattern (e.g. render a fetched entry in a
  page/component), using **propose → confirm before editing the user's components**. The user extends
  the rest from the pattern.
  - **Escalation:** if the user's prompt explicitly asks to replace *all* the hard-coded content
    (not just set up the SDK), do the full wire-up — find every hard-coded spot that maps to CMS
    content and convert it to SDK calls — still **propose → confirm before editing**. Default stays
    the single-example pattern when the ask is just "set up the SDK."
- Use `withContext`, image transforms, and preview-vs-delivery selection where relevant, pulled from
  the SDK README.

### New: `architect-extract`
Analyze a project for hard-coded content and migrate it into the CMS. **propose → confirm → apply.**

**0. Scope prompt (first).** Before scanning, ask the user **how much content** to pull out of the
app — they drive the breadth. Cover:
  - **Breadth:** a single file/component → a directory/feature → the whole app.
  - **Content kinds:** UI strings only, or strings **+ assets** (assets are **optional**, default off
    unless the user opts in).
Use these answers to bound the scan.

1. **Scan** the chosen scope for hard-coded UI strings and (if opted in) asset references (images, etc.).
2. **Propose models** inferred from the patterns; write definitions to local JSON.
3. **Propose entries**; write entry JSON locally.
4. **User reviews/edits** the proposed files.
5. **Apply** only on approval:
   - push models (via `architect-models` flow), then entries (via `architect-entries` flow);
   - if assets were opted in, upload them via the new `architect assets upload` CLI command
     (decision #7) and reference the resulting asset ids in the entry JSON.
- Defers to `architect-models` / `architect-entries` for push mechanics rather than duplicating them.
  MCP-aware via the shared reference.

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

## Cross-repo work (publishing the MCP server — decision #6)

This work spans three repos; the implementation plan must sequence them:

- **`architect-sdk`** — add `packages/mcp` (moved from `architect/services/mcp`), rename scope to
  `@architectcms/mcp`, align `package.json` (version, `bin`/`main`, `files`, `publishConfig: public`),
  and add it to the monorepo build/publish pipeline alongside `cli` and `sdk`. **Also add an
  `architect assets upload` command to `packages/cli`** (decision #7), wrapping the SDK's existing
  `management-assets` upload.
- **`architect` (private)** — remove/deprecate `services/mcp` and update the root `npm run mcp` script
  to consume the published package (or point at the new monorepo location for local dev). Source of
  truth becomes `architect-sdk`.
- **`architect-skills` (this repo)** — the skills, the plugin `.mcp.json`, and the retrofit reference.

The actual `npm publish` is a **manual maintainer step**, not automated by this work.

## Out of scope / dependencies

- **Running `npm publish`** for `@architectcms/mcp` — prepped here, executed by the maintainer.
  `architect-mcp` is non-functional for external users until this happens.
- **Hosted/remote MCP endpoint** — future backend work; only documented as a drop-in here.
- No changes to the CLI or SDK *source* themselves (only `architect-sdk` packaging/pipeline config).

## Versioning

Feature release → bump `marketplace.json` and `plugin.json` to **0.3.0** in sync; mirror in the
Codex plugin manifest. Conventional Commits per skill.

## Open questions

- None outstanding.
