# MCP / SDK / Extract Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `architect-mcp`, `architect-sdk`, and `architect-extract` skills to this repo, make the existing 10 skills MCP-aware, and bundle the Architect MCP server into the Claude plugin.

**Architecture:** All deliverables here are Markdown skills + JSON manifests in the **architect-skills** repo. Skills stay CLI-first; MCP is an optional alternative surfaced via a shared reference. The Claude plugin declares the MCP server through `plugin.json` `mcpServers` + `userConfig` so Claude Code prompts for credentials at enable time. "Tests" in this repo are the CI validations (SKILL.md frontmatter + plugin/marketplace JSON parse) plus content greps — there is no unit-test harness for Markdown.

**Tech Stack:** Markdown (`SKILL.md`), JSON manifests (`.claude-plugin/`, `.codex-plugin/`, `.mcp.json`), Node one-liner validators (mirrors `.github/workflows/validate.yml`).

---

## Prerequisite (SEPARATE PLAN — `architect-sdk` repo, not this one)

The runtime that `architect-mcp` and `architect-extract` rely on does **not exist on public npm yet**. These tasks live in `../architect-sdk` (+ `../architect` cleanup) and need their own plan + a manual `npm publish`:

1. Move `architect/services/mcp` → `architect-sdk/packages/mcp`; rename scope `@architect-cms/mcp` → `@architectcms/mcp`; add `bin`, `files`, `publishConfig: { access: "public" }`; wire into the monorepo build/publish pipeline. **Publish `@architectcms/mcp`.**
2. Add an `architect assets upload <file> [--alt ...]` command to `architect-sdk/packages/cli` wrapping the SDK's existing `ArchitectManagement` → `management-assets.upload` (`POST /api/v2/assets/upload`). **Publish the CLI bump.**
3. `architect` (private): remove/deprecate `services/mcp`, point the root `npm run mcp` script at the monorepo location for local dev.

**This plan's skills are written to reference those published names.** The Markdown ships and passes CI immediately; the MCP server and `assets upload` only become *runnable* for end users once the prerequisite publishes land. Each affected skill notes this. Build this plan independent of the prerequisite; do not block on the publish.

---

## File Structure

**New:**
- `skills/architect-mcp/SKILL.md` — install/configure the MCP server (plugin-bundled + manual + remote).
- `skills/architect-mcp/references/mcp-equivalents.md` — canonical CLI command → MCP tool mapping (shared by all skills; lives here so it ships with `./skills/` to every distribution).
- `skills/architect-sdk/SKILL.md` — install `@architectcms/sdk`, scaffold delivery/preview client, `.env`, fetch + wire content.
- `skills/architect-extract/SKILL.md` — scope-prompt → scan → propose → confirm → apply migration of hard-coded content.
- `.mcp.json` (repo root) — plugin MCP server definition (`npx -y @architectcms/mcp`, env from `${user_config.*}`).

**Modified:**
- `.claude-plugin/plugin.json` — add `mcpServers` + `userConfig`; bump version `0.3.0`.
- `.claude-plugin/marketplace.json` — bump plugin version `0.3.0`.
- `.codex-plugin/plugin.json` — bump version `0.3.0`; refresh description to mention SDK/MCP/extract.
- `package.json` — bump `0.3.0`; add `.mcp.json` to `files`.
- `skills/architect-setup/SKILL.md` — "where your key goes" table + pointers to new skills.
- The 10 existing `skills/*/SKILL.md` — one-line MCP pointer (architect-setup folded into its own task).
- `README.md` — list the 3 new skills.

---

## Task 1: `architect-mcp` skill + shared MCP-equivalents reference

**Files:**
- Create: `skills/architect-mcp/SKILL.md`
- Create: `skills/architect-mcp/references/mcp-equivalents.md`

- [ ] **Step 1: Write the skill file**

Create `skills/architect-mcp/SKILL.md` with this exact frontmatter and the sections below:

```markdown
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

\```json
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
\```

For Claude Desktop, use the same block under `mcpServers` in
`~/Library/Application Support/Claude/claude_desktop_config.json`, but `npx` requires an absolute path or a shell wrapper there.

The key is a **management** key (`arch_mgmt_…`) — the same kind the CLI uses. See `architect-setup`. If you've already run `architect login`, you can **omit the `env` block entirely** — the server falls back to `~/.architect/credentials.json`.

## Option C — Remote (future drop-in)

When a hosted endpoint exists, replace the stdio block with an HTTP one:

\```json
{
  "mcpServers": {
    "architect-cms": {
      "type": "http",
      "url": "https://mcp.architectcms.com",
      "headers": { "Authorization": "Bearer ${ARCHITECT_API_KEY}" }
    }
  }
}
\```

## Verify

In Claude Code: `/mcp` lists `architect-cms` with tools like `list_models`, `search_entries`, `create_<model>`. Ask the agent to "list models" to confirm.

## Local-path fallback (before `@architectcms/mcp` is published)

Clone the source and point at it directly:

\```json
{ "mcpServers": { "architect-cms": {
  "command": "node",
  "args": ["/absolute/path/to/architect-sdk/packages/mcp/index.js"],
  "env": { "ARCHITECT_URL": "…", "ARCHITECT_API_KEY": "…", "ARCHITECT_ORG_ID": "…", "ARCHITECT_ENV_ID": "…" }
}}}
\```

## Related skills

- `architect-setup` — get a management API key and log in the CLI
- Every architect skill — see `references/mcp-equivalents.md` for the CLI↔MCP mapping
```

- [ ] **Step 2: Write the shared MCP-equivalents reference**

Create `skills/architect-mcp/references/mcp-equivalents.md`:

```markdown
# CLI ↔ MCP equivalents

Skills are CLI-first. If the Architect MCP server is installed (`architect-mcp`), these MCP tools do the same work. The CLI remains preferred for scripting, CI, and bulk operations.

## Per-model tools (generated from your schema)

| CLI | MCP tool |
| --- | --- |
| `architect entries pull --model X` | `list_<x>` (paginated), `get_<x>` (single) |
| `architect entries push --model X file.json` (create) | `create_<x>` (one entry, typed fields) |
| `architect entries push --model X file.json` (update) | `update_<x>` (partial, by id) |
| (delete) | `delete_<x>` |

## Static tools

| CLI | MCP tool |
| --- | --- |
| `architect models pull` | `list_models` |
| (full-text search) | `search_entries` |
| (find back-references) | `get_entry_references` |
| `architect env list` | `list_environments` |
| re-login with target env | `switch_environment` |

## Resources

| URI | Description |
| --- | --- |
| `architect://models/{modelId}` | Model definition with fields |
| `architect://entries/{entryId}` | Entry data with metadata |

## Notes

- MCP `create_/update_` operate on **one** entry with typed fields; the CLI `entries push` handles **arrays** (bulk upsert). Use the CLI for bulk.
- MCP has no model-authoring tools — use the CLI (`architect-models`) to create/change schemas.
```

- [ ] **Step 3: Validate frontmatter + content**

Run:
```bash
node -e 'const fs=require("fs");const t=fs.readFileSync("skills/architect-mcp/SKILL.md","utf8");const m=t.match(/^---\n([\s\S]*?)\n---/);if(!m)throw"no frontmatter";if(!/\nname:\s*architect-mcp/.test("\n"+m[1]))throw"bad name";if(!/\ndescription:\s*\S/.test("\n"+m[1]))throw"no description";console.log("ok")'
test -f skills/architect-mcp/references/mcp-equivalents.md && grep -q "search_entries" skills/architect-mcp/references/mcp-equivalents.md && echo "ref ok"
```
Expected: `ok` then `ref ok`.

- [ ] **Step 4: Commit**

```bash
git add skills/architect-mcp
git commit -m "feat: architect-mcp skill + CLI↔MCP equivalents reference"
```

---

## Task 2: Bundle the MCP server in the Claude plugin

**Files:**
- Create: `.mcp.json`
- Modify: `.claude-plugin/plugin.json`
- Modify: `package.json` (add `.mcp.json` to `files`)

- [ ] **Step 1: Create the repo-root `.mcp.json`**

```json
{
  "mcpServers": {
    "architect-cms": {
      "command": "npx",
      "args": ["-y", "@architectcms/mcp"],
      "env": {
        "ARCHITECT_URL": "${user_config.architect_url}",
        "ARCHITECT_API_KEY": "${user_config.architect_api_key}",
        "ARCHITECT_ORG_ID": "${user_config.architect_org_id}",
        "ARCHITECT_ENV_ID": "${user_config.architect_env_id}"
      }
    }
  }
}
```

- [ ] **Step 2: Add `mcpServers` + `userConfig` to `.claude-plugin/plugin.json`**

Add these keys to the existing object (keep current fields; bump version in Task 6):

```json
  "mcpServers": "./.mcp.json",
  "userConfig": {
    "architect_url": {
      "type": "string",
      "title": "Architect API URL",
      "description": "Base URL of your Architect API.",
      "default": "https://api.architectcms.com"
    },
    "architect_api_key": {
      "type": "string",
      "title": "Management API key (optional)",
      "description": "Optional — leave blank to reuse your `architect login` credentials. Set an arch_mgmt_… key only to override the logged-in account.",
      "sensitive": true,
      "required": false
    },
    "architect_org_id": {
      "type": "string",
      "title": "Organization ID (optional)",
      "description": "Optional — leave blank to reuse `architect login`. Override with an org_… id if needed.",
      "required": false
    },
    "architect_env_id": {
      "type": "string",
      "title": "Environment ID (optional)",
      "description": "Optional — leave blank to reuse `architect login`. Override with an env_… id. Switchable at runtime.",
      "required": false
    }
  }
```

Note: every field is `required: false`. When blank, the env var is empty and the MCP server falls back to the CLI's `~/.architect/credentials.json` (see decision in the SDK plan's `config.js` change). So plugin users who already ran `architect login` get a working MCP server with zero extra input, and CLI-only users are never blocked.

- [ ] **Step 3: Add `.mcp.json` to `package.json` `files`**

Change `"files": [".claude-plugin", "skills", "README.md", "LICENSE"]` to include `.mcp.json`:
```json
  "files": [".claude-plugin", "skills", ".mcp.json", "README.md", "LICENSE"],
```

- [ ] **Step 4: Validate the JSON**

Run:
```bash
node -e 'JSON.parse(require("fs").readFileSync(".mcp.json","utf8"));const p=JSON.parse(require("fs").readFileSync(".claude-plugin/plugin.json","utf8"));if(p.mcpServers!=="./.mcp.json")throw"mcpServers not wired";if(!p.userConfig||!p.userConfig.architect_api_key)throw"userConfig missing";console.log("plugin mcp ok")'
```
Expected: `plugin mcp ok`.

- [ ] **Step 5: Commit**

```bash
git add .mcp.json .claude-plugin/plugin.json package.json
git commit -m "feat: bundle Architect MCP server in the Claude plugin via userConfig"
```

---

## Task 3: `architect-sdk` skill

**Files:**
- Create: `skills/architect-sdk/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `skills/architect-sdk/SKILL.md`:

```markdown
---
name: architect-sdk
description: Set up the Architect CMS SDK (@architectcms/sdk) in an application — install it, scaffold a delivery and/or preview client with an example .env, then fetch content from the CMS and wire it into the app. Use when the user wants their app to read content from Architect.
---

# architect-sdk

Wire `@architectcms/sdk` into an app and pull CMS content into it. This is the inverse of `architect-extract` (which pushes hard-coded content up to the CMS).

## 1. Install

\```bash
npm install @architectcms/sdk
\```

## 2. Pick the client + key

| Client | Key prefix | Use |
| --- | --- | --- |
| `ArchitectDelivery` | `arch_delivery_…` | Published content (production) |
| `ArchitectPreview` | `arch_preview_…` | Drafts / unpublished (preview builds) |

`ArchitectManagement` (`arch_mgmt_…`) also exists but app code should prefer delivery/preview.

## 3. Example `.env`

Create `.env.example` (and ensure `.env` is git-ignored):

\```
ARCHITECT_ORG_ID=org_…
ARCHITECT_ENV_ID=env_…
ARCHITECT_DELIVERY_KEY=arch_delivery_…
ARCHITECT_PREVIEW_KEY=arch_preview_…
\```

\```bash
grep -qxF '.env' .gitignore || echo '.env' >> .gitignore
\```

## 4. Typed data-fetching layer

First generate types (see `architect-types`: `architect types generate`), then a small client module:

\```ts
// lib/architect.ts
import { ArchitectDelivery, ArchitectPreview } from '@architectcms/sdk'

const base = {
  organizationId: process.env.ARCHITECT_ORG_ID!,
  environmentId: process.env.ARCHITECT_ENV_ID!,
  baseUrl: 'https://api.architectcms.com',
}

export const cms =
  process.env.ARCHITECT_PREVIEW === '1'
    ? new ArchitectPreview({ ...base, apiKey: process.env.ARCHITECT_PREVIEW_KEY! })
    : new ArchitectDelivery({ ...base, apiKey: process.env.ARCHITECT_DELIVERY_KEY! })

export const getArticles = () => cms.entries.model('article').fetch()
export const getArticle = (id: string) => cms.entries.get(id)
\```

## 5. Verify the fetch

Before editing UI, confirm content comes back:

\```bash
node --env-file=.env -e "import('@architectcms/sdk').then(async ({ArchitectDelivery})=>{const c=new ArchitectDelivery({organizationId:process.env.ARCHITECT_ORG_ID,environmentId:process.env.ARCHITECT_ENV_ID,apiKey:process.env.ARCHITECT_DELIVERY_KEY,baseUrl:'https://api.architectcms.com'});console.log((await c.entries.model('article').fetch()).length+' entries')})"
\```

## 6. Hook it up (propose → confirm before editing components)

- **Default:** wire **one** representative spot end-to-end as a copyable pattern (detect the framework — Next.js server component, Remix loader, plain React fetch, etc. — and follow its convention). Show the diff and get confirmation before editing the user's files. The user extends the rest.
- **Escalation — replace ALL hard-coded content:** only when the user explicitly asks to replace all hard-coded content, find every hard-coded spot that maps to CMS content and convert it to SDK calls. Still **propose the full set of edits and confirm before applying.**

Use `withContext(...)` for context-aware content and `cms.assets.imageUrl(id, { width, format })` for images, per the SDK README.

## Related skills

- `architect-types` — generate types the data layer uses
- `architect-extract` — push hard-coded content up to the CMS (the inverse direction)
- `architect-setup` — where each kind of key lives
```

- [ ] **Step 2: Validate**

Run:
```bash
node -e 'const fs=require("fs");const t=fs.readFileSync("skills/architect-sdk/SKILL.md","utf8");const m=t.match(/^---\n([\s\S]*?)\n---/);if(!m)throw"no frontmatter";if(!/\nname:\s*architect-sdk/.test("\n"+m[1]))throw"bad name";if(!/arch_delivery_/.test(t)||!/arch_preview_/.test(t))throw"missing key prefixes";console.log("ok")'
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add skills/architect-sdk
git commit -m "feat: architect-sdk skill (install, .env, fetch + wire content)"
```

---

## Task 4: `architect-extract` skill

**Files:**
- Create: `skills/architect-extract/SKILL.md`

- [ ] **Step 1: Write the skill file**

Create `skills/architect-extract/SKILL.md`:

```markdown
---
name: architect-extract
description: Analyze an application for hard-coded strings and assets, model that content, and migrate it into Architect CMS via the architect CLI. Prompts for how much content to extract; proposes models and entries for review before pushing. Use when the user wants to move hard-coded content into the CMS.
---

# architect-extract

Find hard-coded content in an app, model it, and move it into Architect. **Propose → confirm → apply** — nothing is pushed until you approve. This is the inverse of `architect-sdk` (which pulls content back into the app).

## Prerequisites

`architect-setup` done (CLI logged in with a management key). Asset upload needs the `architect assets upload` command (CLI ≥ the version that ships it).

## 0. Ask how much content to extract

Before scanning, agree on scope with the user:

- **Breadth:** a single file/component → a directory/feature → the whole app.
- **Kinds:** UI strings only, or **strings + assets** (assets default **off** unless the user opts in).

Use the answers to bound everything below.

## 1. Scan

Within the chosen scope, find hard-coded UI strings (JSX text, label/title/alt attributes, string constants) and, if assets were opted in, referenced image/file paths. Group related strings that look like one logical record (e.g. a card's title + body + image) — those become one model.

## 2. Propose models (local, for review)

Write inferred model definitions to `architect/extract/models.json` in the shape `architect-models` expects. Do **not** push yet. Show the user the proposed models and let them rename/trim fields.

## 3. Propose entries (local, for review)

Write entry data to `architect/extract/entries.<model>.json` in the `architect-entries` upsert shape (array of `{ "data": { … } }`, no ids for new content). For assets, leave a placeholder field to be filled with the uploaded asset id in step 5.

## 4. Confirm

Summarize: N models, M entries, K assets. Get explicit approval before any push.

## 5. Apply (only after approval)

1. Push models — see `architect-models` (`architect models push architect/extract/models.json`).
2. If assets were opted in, upload each and capture its id:
   \```bash
   architect assets upload ./path/to/image.png --alt "…"
   \```
   Write the returned asset id into the relevant entry field.
3. Push entries — see `architect-entries` (`architect entries push --model X architect/extract/entries.X.json`).

## 6. Next step

Offer to run `architect-sdk` to wire the now-CMS-managed content back into the app, replacing the hard-coded originals.

## Notes

- Defers to `architect-models` and `architect-entries` for push mechanics — don't reimplement them.
- MCP installed? The same operations map to MCP tools — see `architect-mcp/references/mcp-equivalents.md`. The CLI is still preferred for bulk upsert.

## Related skills

- `architect-models`, `architect-entries` — the push targets
- `architect-sdk` — pull the migrated content back into the app
```

- [ ] **Step 2: Validate**

Run:
```bash
node -e 'const fs=require("fs");const t=fs.readFileSync("skills/architect-extract/SKILL.md","utf8");const m=t.match(/^---\n([\s\S]*?)\n---/);if(!m)throw"no frontmatter";if(!/\nname:\s*architect-extract/.test("\n"+m[1]))throw"bad name";if(!/assets upload/.test(t))throw"missing asset upload";console.log("ok")'
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
git add skills/architect-extract
git commit -m "feat: architect-extract skill (scan, propose, confirm, migrate)"
```

---

## Task 5: Update `architect-setup` + retrofit MCP pointers into existing skills

**Files:**
- Modify: `skills/architect-setup/SKILL.md`
- Modify: `skills/architect-{models,entries,types,localization,context,context-actions,lifecycle,webhooks,env}/SKILL.md` (9 files)

- [ ] **Step 1: Add the key-location table + pointers to `architect-setup`**

In `skills/architect-setup/SKILL.md`, after the "Confirm" section, add:

```markdown
## Where your API key goes

Different surfaces use different keys in different places:

| Use | Key type | Location |
| --- | --- | --- |
| CLI | `arch_mgmt_…` | `~/.architect/credentials.json` via `architect login` |
| MCP server | `arch_mgmt_…` | **Reuses the CLI's `~/.architect/credentials.json` automatically.** Override via plugin `userConfig` or `.mcp.json` env only if needed — see `architect-mcp` |
| SDK (delivery/preview) | `arch_delivery_…` / `arch_preview_…` | project `.env` — see `architect-sdk` |

`architect login` configures both the CLI **and** the MCP server — you only set the management key once. Never commit a key. The CLI stores yours at `~/.architect/credentials.json` (0600).
```

Then update the "Related skills" list to also reference `architect-sdk`, `architect-mcp`, and `architect-extract`.

- [ ] **Step 2: Add the one-line MCP pointer to each of the 9 remaining skills**

In each of the 9 files, add this line to the "Related skills" section (or create one at the end if absent):

```markdown
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
```

- [ ] **Step 3: Validate all skills still pass frontmatter + pointer present**

Run (mirrors CI, then checks the pointer landed in all 10 non-mcp/sdk/extract skills):
```bash
node -e '
const fs=require("fs"),path=require("path");const dir="skills";let bad=0;
for(const n of fs.readdirSync(dir)){const f=path.join(dir,n,"SKILL.md");const t=fs.readFileSync(f,"utf8");const m=t.match(/^---\n([\s\S]*?)\n---/);if(!m||!/\nname:\s*\S/.test("\n"+m[1])||!/\ndescription:\s*\S/.test("\n"+m[1])){console.error("bad",f);bad++;}}
for(const n of ["architect-setup","architect-models","architect-entries","architect-types","architect-localization","architect-context","architect-context-actions","architect-lifecycle","architect-webhooks","architect-env"]){const t=fs.readFileSync("skills/"+n+"/SKILL.md","utf8");if(!/architect-mcp/.test(t)){console.error("no mcp pointer",n);bad++;}}
process.exit(bad?1:0);'
echo "validated"
```
Expected: `validated` with no error lines.

- [ ] **Step 4: Commit**

```bash
git add skills
git commit -m "docs: key-location table in architect-setup; MCP pointers across skills"
```

---

## Task 6: Version bump + README + final validation

**Files:**
- Modify: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json`, `package.json`
- Modify: `README.md`

- [ ] **Step 1: Bump all versions to `0.3.0`**

Set `"version": "0.3.0"` in `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` (the plugin entry), `.codex-plugin/plugin.json`, and `package.json`. Refresh the `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, and `.codex-plugin/plugin.json` descriptions to mention SDK setup, MCP, and content extraction.

- [ ] **Step 2: Add the 3 new skills to `README.md`**

Add `architect-mcp`, `architect-sdk`, and `architect-extract` to the skills list in `README.md` with one-line summaries matching their frontmatter descriptions.

- [ ] **Step 3: Run the full CI validation locally**

Run (the exact two checks from `.github/workflows/validate.yml`, plus the new `.mcp.json`):
```bash
node -e '
const fs=require("fs"),path=require("path");const dir="skills";let bad=0;
for(const name of fs.readdirSync(dir)){const f=path.join(dir,name,"SKILL.md");if(!fs.existsSync(f)){console.error("missing",f);bad++;continue;}const t=fs.readFileSync(f,"utf8");const m=t.match(/^---\n([\s\S]*?)\n---/);if(!m){console.error("no frontmatter",f);bad++;continue;}if(!/\nname:\s*\S/.test("\n"+m[1])){console.error("no name",f);bad++;}if(!/\ndescription:\s*\S/.test("\n"+m[1])){console.error("no description",f);bad++;}}
process.exit(bad?1:0);'
node -e 'JSON.parse(require("fs").readFileSync(".claude-plugin/plugin.json","utf8"));JSON.parse(require("fs").readFileSync(".claude-plugin/marketplace.json","utf8"));JSON.parse(require("fs").readFileSync(".codex-plugin/plugin.json","utf8"));JSON.parse(require("fs").readFileSync(".mcp.json","utf8"));console.log("json ok")'
```
Expected: exit 0 (no error lines) then `json ok`.

- [ ] **Step 4: Verify all four manifests agree on version**

Run:
```bash
node -e 'const f=s=>JSON.parse(require("fs").readFileSync(s,"utf8"));const v=[f(".claude-plugin/plugin.json").version,f(".claude-plugin/marketplace.json").plugins[0].version,f(".codex-plugin/plugin.json").version,f("package.json").version];if(new Set(v).size!==1||v[0]!=="0.3.0")throw"version mismatch: "+v;console.log("versions ok 0.3.0")'
```
Expected: `versions ok 0.3.0`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "chore: v0.3.0 — architect-mcp, architect-sdk, architect-extract skills"
```

---

## Self-Review notes

- **Spec coverage:** architect-mcp (Tasks 1–2), architect-sdk incl. fetch/wire + escalation (Task 3), architect-extract incl. scope prompt + optional assets (Task 4), key-location table + MCP retrofit (Task 5), versioning (Task 6). Cross-repo publish + `assets upload` command are explicitly the separate prerequisite plan.
- **Dependency honesty:** `@architectcms/mcp` and `architect assets upload` are unpublished; skills note this and the local-path fallback. Markdown ships and passes CI regardless.
- **Naming consistency:** userConfig keys `architect_url` / `architect_api_key` / `architect_org_id` / `architect_env_id` are identical in `.mcp.json` and `plugin.json`. Env var names `ARCHITECT_URL/API_KEY/ORG_ID/ENV_ID` match the MCP server's `config.js`.
```
