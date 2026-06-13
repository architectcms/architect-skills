# Architect Skills — Agent Skills & Multi-Platform Marketplace Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the public `architectcms/architect-skills` repo — a set of CLI-driven Agent Skills that teach Claude, Codex, Cursor, and other terminal agents how to manage Architect CMS (model content, context models/providers, lifecycle functions, webhooks, environments, types) entirely through the `architect` CLI — packaged so it installs as a Claude Code plugin marketplace, a Codex skill set, and via the `npx skills` installer (listed on skills.sh).

**Architecture:** One public repo is simultaneously (a) the canonical skills source at `skills/<name>/SKILL.md` (what skills.sh + Codex + Cursor discover), and (b) a Claude Code plugin marketplace via root `.claude-plugin/{marketplace.json,plugin.json}` whose single plugin's skills dir points at that same `skills/`. Every skill is **CLI-driven** — it instructs the agent to run `architect …` commands (from `@architectcms/cli`), never raw HTTP. This supersedes the curl-based skills in `../architect/skills`.

**Tech Stack:** Markdown (Agent Skills `SKILL.md` standard with YAML frontmatter), JSON manifests (Claude plugin/marketplace schema), `@architectcms/cli` (the runtime the skills drive), GitHub Actions for validation. No application code.

---

## ⚠️ Prerequisite (resolve before Phases 2–4)

The CLI-driven skills assume the CLI can perform every operation. Today `@architectcms/cli@0.1.1` only ships `login`, `whoami`, `logout`, `models`, `entries`, `types`, `init`. The commands `contexts`, `context-actions`, `lifecycle`, `webhooks`, `env` are added by the companion plan **`architect-sdk/docs/plans/2026-06-12-management-expansion.md`** (which also adds the underlying SDK resources, since the CLI does no direct HTTP).

**Dependency map** — which skill needs which CLI command:

| Skill | CLI commands used | Available today? |
| --- | --- | --- |
| `architect-setup` | `login`, `whoami` | ✅ |
| `architect-models` | `models pull/push` | ✅ |
| `architect-entries` | `entries pull/push` | ✅ |
| `architect-types` | `types generate` | ✅ |
| `architect-localization` | `init --config` | ✅ |
| `architect-context` | `contexts list/pull/push` | ⛔ needs companion plan |
| `architect-context-actions` | `context-actions list/run` | ⛔ needs companion plan |
| `architect-lifecycle` | `lifecycle list/add/rm` | ⛔ needs companion plan |
| `architect-webhooks` | `webhooks list/add/test/rm` | ⛔ needs companion plan |
| `architect-env` | `env list/create` | ⛔ needs companion plan |

**Phase 1 ships the ✅ skills now** (they work against the published `@architectcms/cli@0.1.1`). **Phase 4 adds the ⛔ skills** and is gated on the companion plan shipping a CLI version (call it `>=0.2.0`) that includes those commands. Pin the README/skills to that minimum CLI version.

---

## Source material

Each skill is a **CLI rewrite** of the existing curl-based skill at `/Users/nkranendonk/Projects/architect/skills/<name>/SKILL.md` (and its `references/*.md`). Port the conceptual content (field types, query syntax, lifecycle examples, context strategies) verbatim where useful, but **replace every `curl … $API_URL …` block with the equivalent `architect …` command** and **delete the manual `~/.architect/config.json` + `jq` configuration block** (the CLI owns auth/config now). The exact curl→CLI mapping is given per skill below.

---

## Repo layout (target)

```
architect-skills/
├── .claude-plugin/
│   ├── marketplace.json          # marketplace catalog (lists one plugin: source ".")
│   └── plugin.json               # the plugin manifest; skills at ./skills
├── skills/
│   ├── architect-setup/SKILL.md
│   ├── architect-models/SKILL.md          (+ references/field-types.md)
│   ├── architect-entries/SKILL.md         (+ references/query-syntax.md)
│   ├── architect-types/SKILL.md
│   ├── architect-localization/SKILL.md    (+ examples/init.fr-en.json)
│   ├── architect-context/SKILL.md         (+ references/strategies.md)   # Phase 4
│   ├── architect-context-actions/SKILL.md                               # Phase 4
│   ├── architect-lifecycle/SKILL.md       (+ references/examples.md)     # Phase 4
│   ├── architect-webhooks/SKILL.md                                      # Phase 4
│   └── architect-env/SKILL.md                                           # Phase 4
├── package.json                  # @architectcms/skills (metadata for skills.sh/npm)
├── README.md                     # multi-platform install (Claude / Codex / skills.sh / Cursor)
├── LICENSE                       # MIT
├── CLAUDE.md                     # repo-hygiene guardrails (public repo)
├── .gitignore
├── .github/workflows/validate.yml  # frontmatter + plugin validation
└── docs/plans/2026-06-12-architect-skills-marketplace.md  (this file)
```

**Why one repo, not two (skills repo + marketplace repo):** a Claude marketplace is just a `marketplace.json` manifest; it can list a plugin whose `source` is `.` (the repo root). So this single repo *is* the marketplace, *is* the plugin, and *is* the skills source for skills.sh/Codex. Split out a dedicated marketplace repo only if you later host third-party/community plugins you don't own — not needed for v1.

**SKILL.md frontmatter (required for Claude/Codex/skills.sh):** every `SKILL.md` starts with YAML frontmatter — `name` (kebab-case, = directory name) and `description` (what it does + when to use; `description` is capped at 1,536 chars). The existing architect skills have **no** frontmatter (they start with an H1) — adding it is mandatory for these to work as Agent Skills.

---

## Phase 0 — Repo scaffold

### Task 0.1: Initialize the repo

**Files:** Create `package.json`, `LICENSE`, `.gitignore`, `CLAUDE.md`, `README.md` (stub).

- [ ] **Step 1: `git init` and base files**

```bash
cd /Users/nkranendonk/Projects/architect-skills
git init -b main
printf 'node_modules/\n.DS_Store\n.playwright-mcp/\n*.log\n' > .gitignore
```

- [ ] **Step 2: Write `package.json`** (metadata only — no build; `files` ships skills + manifests):

```json
{
  "name": "@architectcms/skills",
  "version": "0.1.0",
  "description": "Agent Skills for managing Architect CMS through the architect CLI (Claude, Codex, Cursor, and more).",
  "license": "MIT",
  "type": "module",
  "private": false,
  "keywords": ["architect", "architectcms", "cms", "agent-skills", "claude", "codex", "skills", "headless"],
  "repository": { "type": "git", "url": "git+https://github.com/architectcms/architect-skills.git" },
  "homepage": "https://github.com/architectcms/architect-skills#readme",
  "bugs": "https://github.com/architectcms/architect-skills/issues",
  "files": [".claude-plugin", "skills", "README.md", "LICENSE"],
  "publishConfig": { "access": "public" }
}
```

- [ ] **Step 3: Write `LICENSE`** — MIT, copyright "Architect CMS". (Copy an MIT template; year 2026.)

- [ ] **Step 4: Write `CLAUDE.md`** (this is a public repo — same hygiene rules as architect-sdk):

```markdown
# architect-skills

Public repo of CLI-driven Agent Skills for [Architect CMS](https://architectcms.com). Each skill teaches a terminal agent (Claude Code, Codex, Cursor, …) to manage content via the `architect` CLI (`@architectcms/cli`). Distributed as a Claude plugin marketplace, Codex skills, and via `npx skills`.

## Repository hygiene — READ FIRST
This repository is PUBLIC; its full git history is visible. Never commit:
- Secrets — API keys (`arch_*`), tokens, passwords, `.env`, `~/.architect/credentials.json`. Skills must tell users to run `architect login`, never embed a key. Use placeholders (`arch_mgmt_…`) in examples.
- Internal planning/design docs beyond `docs/plans/`, or anything describing private backend internals.
- Local/agent artifacts (`.playwright-mcp/`, scratch files).

## Conventions
- Every skill lives at `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`).
- Skills are CLI-driven: they instruct the agent to run `architect …`. No raw HTTP/curl.
- Conventional Commits. Keep `marketplace.json`/`plugin.json` versions in sync with releases.
```

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: scaffold architect-skills repo"
```

### Task 0.2: Create the GitHub repo and push

- [ ] **Step 1: Create the public repo and push** (gh is authed):

```bash
gh repo create architectcms/architect-skills --public --source=. --remote=origin --description "Agent Skills for managing Architect CMS via the architect CLI" --push
```
Expected: repo created at `github.com/architectcms/architect-skills`, `main` pushed.

- [ ] **Step 2: Confirm** `gh repo view architectcms/architect-skills --json visibility -q .visibility` → `PUBLIC`.

---

## Phase 1 — Ship-now skills (work with `@architectcms/cli@0.1.1`)

> Skill authoring pattern (match across all skills): YAML frontmatter (`name`, `description`) → `## Overview` → `## Prerequisites` (must have run `architect login`) → `## Commands` (each operation = an `architect …` invocation with a short example + expected output) → `## Workflows` (multi-step recipes) → `## Tips` → `## Related skills`. Keep ≤ ~300 lines; push deep tables to `references/*.md`. Pull conceptual content from the matching `../architect/skills/<name>` source, converting curl → CLI.

### Task 1.1: `architect-setup` skill

**Files:** Create `skills/architect-setup/SKILL.md`.

The single biggest simplification: the old `setup` skill did `curl` register/login/key-mint + wrote `~/.architect/config.json`+`config.local.json` by hand. The CLI replaces all of it with `architect login`.

- [ ] **Step 1: Write `skills/architect-setup/SKILL.md`:**

```markdown
---
name: architect-setup
description: Set up the Architect CMS CLI — install @architectcms/cli, log in with a management API key, and confirm the connection. Use this first, before any other architect skill, or whenever architect commands fail with "Not logged in".
---

# architect-setup

Get a terminal agent ready to manage Architect CMS via the `architect` CLI.

## Overview
All other architect skills drive the `architect` CLI. This skill installs it and authenticates. Auth uses a **management API key** (`arch_mgmt_…`) created in the Architect web app — the CLI stores it at `~/.architect/credentials.json` (0600).

## Install the CLI
```bash
npm install -g @architectcms/cli   # provides the `architect` command
architect --version
```

## Log in
Interactive (prompts for key, org id, env id):
```bash
architect login
```
Non-interactive / CI (pass everything as flags — never paste a key into a file the agent commits):
```bash
architect login --api-key "$ARCHITECT_API_KEY" --organization-id "$ARCHITECT_ORG" --environment-id "$ARCHITECT_ENV"
```
Where to get these: create a **management** API key in the web app under Settings → API Keys; the org id and environment id are shown alongside it.

## Confirm
```bash
architect whoami     # prints org / env / baseUrl and "(key valid)"
```
If `whoami` says "Not logged in", run `architect login` again.

## Tips
- Self-hosted? Pass `--base-url https://your-host` to `login` (default is `https://api.architectcms.com`).
- `architect logout` clears stored credentials.

## Related skills
- `architect-models` — define content models
- `architect-entries` — create/update content
```

- [ ] **Step 2: Commit** (`feat: architect-setup skill`).

### Task 1.2: `architect-models` skill

**Files:** Create `skills/architect-models/SKILL.md`, `skills/architect-models/references/field-types.md`.

curl→CLI mapping: list/get → `architect models pull`; create/update → edit the pulled JSON and `architect models push <file>`. Context **models** are created here too (a model with `isContextModel: true` and a `keyField`).

- [ ] **Step 1: Write `skills/architect-models/SKILL.md`:**

```markdown
---
name: architect-models
description: Define and evolve Architect CMS content models (schemas) — create models and fields, set up context source models, and pull/push model definitions as JSON via the architect CLI. Use when the user wants to model content, add fields, or change a schema.
---

# architect-models

Manage content models (schemas) with the `architect` CLI.

## Prerequisites
Run `architect-setup` first (`architect whoami` must show a valid key).

## Pull the current models
```bash
architect models pull --out architect/models.json   # all models as JSON
```

## Create or update models
Models are managed as a JSON array. Pull, edit, push — push creates models that don't exist and updates those matched by id or name:
```bash
architect models push architect/models.json
```
A model object:
```json
{
  "name": "Article",
  "displayName": "Article",
  "description": "A blog article",
  "fields": [
    { "name": "title", "type": "text", "required": true },
    { "name": "body", "type": "richtext" },
    { "name": "author", "type": "model", "targetModelIds": ["author"], "multiple": false }
  ]
}
```

## Context source models
A model used by a context provider is marked `isContextModel: true` and declares a `keyField` (the field whose value selects a context — e.g. a `code` field). Create the model first, then add a self-referencing `parent` relation if it needs a hierarchy (see `architect-context`):
```json
{ "name": "Locale", "isContextModel": true, "keyField": "code",
  "fields": [ { "name": "code", "type": "text", "required": true }, { "name": "name", "type": "text", "required": true } ] }
```

## Field types
See `references/field-types.md` for every field type (`text`, `richtext`, `number`, `boolean`, `date`, `model` (relations), `select`, `key`, `json`, `array`, …) and their options.

## Workflows
- **Add a field:** `architect models pull --out m.json` → add the field object to the model's `fields` → `architect models push m.json`.
- **Generate types after a schema change:** run the `architect-types` skill.

## Related skills
- `architect-entries`, `architect-context`, `architect-types`
```

- [ ] **Step 2: Write `references/field-types.md`** — port the field-type table from `/Users/nkranendonk/Projects/architect/skills/models/references/field-types.md`, but present each as a JSON field object (not a curl body). Keep the canonical `type` enum: `text, richtext, textarea, email, url, string, model, relation, file, image, number, boolean, date, json, array, key, select`.
- [ ] **Step 3: Commit** (`feat: architect-models skill`).

### Task 1.3: `architect-entries` skill

**Files:** Create `skills/architect-entries/SKILL.md`, `skills/architect-entries/references/query-syntax.md`.

curl→CLI mapping: list/fetch → `architect entries pull --model <name>`; create/update → `architect entries push --model <name> <file>` (entries with an `id` update; without create).

- [ ] **Step 1: Write `skills/architect-entries/SKILL.md`:**

```markdown
---
name: architect-entries
description: Create, read, and update Architect CMS content entries via the architect CLI — pull entries for a model to JSON, edit, and push them back (upsert). Use when the user wants to add, edit, bulk-import, or export content.
---

# architect-entries

Manage content entries with the `architect` CLI.

## Prerequisites
`architect-setup` done, and the target model exists (`architect-models`).

## Pull entries
```bash
architect entries pull --model Article --out architect/articles.json
```

## Create / update (upsert)
Push a JSON array. Each item with an `id` is updated; items without an `id` are created. Field values live under `data`:
```json
[
  { "id": "existing_entry_id", "data": { "title": "Updated title" } },
  { "data": { "title": "Brand new article", "body": "<p>Hello</p>" } }
]
```
```bash
architect entries push --model Article architect/articles.json
```

## Relation values
For a single `model`-type relation, set the field to the target entry id; for a multiple relation, an array of ids:
```json
{ "data": { "title": "Post", "author": "author_entry_id", "tags": ["t1", "t2"] } }
```

## Workflows
- **Bulk import:** build the JSON array (no `id`s) → `architect entries push --model X file.json`.
- **Export/migrate:** `architect entries pull --model X --out X.json` from one env, `architect entries push` into another (re-login with the target env first).

## Notes
- `--model` accepts the model name or id; the CLI resolves it.
- See `references/query-syntax.md` for how filtering/relation-resolution works server-side.

## Related skills
- `architect-models`, `architect-context` (locale/segment-aware content)
```

- [ ] **Step 2: Write `references/query-syntax.md`** — port from `/Users/nkranendonk/Projects/architect/skills/entries/references/query-syntax.md` (filter operators, `resolveRelations`), framed as notes on what `entries pull` returns.
- [ ] **Step 3: Commit** (`feat: architect-entries skill`).

### Task 1.4: `architect-types` skill

**Files:** Create `skills/architect-types/SKILL.md`.

- [ ] **Step 1: Write `skills/architect-types/SKILL.md`:**

```markdown
---
name: architect-types
description: Generate TypeScript types for your Architect CMS content models via the architect CLI, so application code is type-safe against the schema. Use after creating or changing models, or when the user wants typed content.
---

# architect-types

Generate TypeScript interfaces from your content models.

## Prerequisites
`architect-setup` done.

## Generate
```bash
architect types generate --output ./architect-types.ts
```
This fetches every model in the current environment and writes one `export interface` per model (fields typed from the schema; relations typed by target).

## Workflow
1. Change schema (`architect-models`).
2. Regenerate: `architect types generate --output src/architect-types.ts`.
3. Import the interfaces in app code.

## Related skills
- `architect-models`
```

- [ ] **Step 2: Commit** (`feat: architect-types skill`).

### Task 1.5: `architect-localization` skill

**Files:** Create `skills/architect-localization/SKILL.md`, `skills/architect-localization/examples/init.fr-en.json`.

Uses `architect init --config` (available today) to scaffold a Locale context model + locale entries + a Localization context provider.

- [ ] **Step 1: Copy the example config** from `/Users/nkranendonk/Projects/architect-sdk/packages/cli/examples/init.fr-en.json` to `skills/architect-localization/examples/init.fr-en.json`.

- [ ] **Step 2: Write `skills/architect-localization/SKILL.md`:**

```markdown
---
name: architect-localization
description: Set up localization in Architect CMS — a Locale context model, seeded locale entries, and a Localization context provider with language→region fallback — via `architect init`. Use when the user wants multi-locale/translated content.
---

# architect-localization

Scaffold localization with one command via `architect init`.

## Prerequisites
`architect-setup` done.

## Interactive
```bash
architect init        # answer the localization prompts (locales, default, hierarchy)
```

## From a config (repeatable / CI)
```bash
architect init --config init.json --yes
```
Example `init.json` (also in `examples/init.fr-en.json`):
```json
{
  "localization": {
    "enabled": true,
    "modelName": "Locale",
    "providerName": "Localization",
    "locales": [
      { "code": "en-US", "name": "English (US)" },
      { "code": "fr-FR", "name": "French (France)" }
    ],
    "defaultLocale": "en",
    "hierarchy": true
  }
}
```
With `hierarchy: true`, a region like `fr-FR` falls back to its base language `fr` (created automatically), then to the default. Resolution is most-specific-first.

## What it creates
- A `Locale` context model (key field `code`).
- One locale entry per code (base languages first, regions linked to them).
- A `Localization` context provider (`derivationPath: []`, `hierarchyRelations: ["parent"]`).

## Related skills
- `architect-context` (general context providers), `architect-models`
```

- [ ] **Step 3: Commit** (`feat: architect-localization skill`).

### Task 1.6: Wire the Claude plugin + marketplace manifests (ship-now skills)

**Files:** Create `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`.

- [ ] **Step 1: Write `.claude-plugin/plugin.json`:**

```json
{
  "name": "architect-cms",
  "description": "Manage Architect CMS from your terminal agent — models, entries, types, localization, context, lifecycle, webhooks, and environments via the architect CLI.",
  "version": "0.1.0",
  "author": { "name": "Architect CMS" },
  "homepage": "https://github.com/architectcms/architect-skills",
  "repository": "https://github.com/architectcms/architect-skills",
  "license": "MIT",
  "keywords": ["architect", "cms", "content", "headless"]
}
```
(Skills are auto-discovered from the root `skills/` directory; the plugin's skills path defaults to `skills`.)

- [ ] **Step 2: Write `.claude-plugin/marketplace.json`:**

```json
{
  "$schema": "https://code.claude.com/schemas/marketplace-v1.json",
  "name": "architect-skills",
  "description": "Agent Skills for managing Architect CMS via the architect CLI.",
  "owner": { "name": "Architect CMS" },
  "plugins": [
    {
      "name": "architect-cms",
      "source": ".",
      "description": "Models, entries, types, localization, context, lifecycle, webhooks, and environments for Architect CMS.",
      "version": "0.1.0",
      "license": "MIT"
    }
  ]
}
```

- [ ] **Step 3: Validate the plugin/marketplace** (Claude Code CLI):

```bash
claude plugin validate .
```
Expected: validates `plugin.json`, `marketplace.json`, and each `SKILL.md` frontmatter with no errors. Fix any frontmatter issues it reports (missing `name`/`description`, description > 1,536 chars).

- [ ] **Step 4: Live install test** in a throwaway dir:

```bash
claude plugin marketplace add architectcms/architect-skills
claude plugin install architect-cms@architect-skills
claude plugin list   # architect-cms shows installed; skills appear as /architect-cms:architect-models etc.
```

- [ ] **Step 5: Commit** (`feat: claude plugin + marketplace manifests`).

---

## Phase 2 — Distribution docs & validation

### Task 2.1: README with multi-platform install

**Files:** Create `README.md`.

- [ ] **Step 1: Write `README.md`** covering all four install paths (these are the verified commands):

````markdown
# Architect CMS — Agent Skills

CLI-driven [Agent Skills](https://agentskills.io) that let Claude, Codex, Cursor, and other terminal agents manage [Architect CMS](https://architectcms.com): model content, manage entries, generate types, set up localization, context providers, lifecycle functions, webhooks, and environments — all through the `architect` CLI.

> **Requires** the CLI: `npm i -g @architectcms/cli` (>= 0.2.0 for the context/lifecycle/webhooks/env skills). Run `architect login` once. See the `architect-setup` skill.

## Install

### Claude Code (plugin marketplace)
```
/plugin marketplace add architectcms/architect-skills
/plugin install architect-cms@architect-skills
```

### Codex
```
$skill-installer install https://github.com/architectcms/architect-skills/tree/main/skills/architect-models
```
(repeat per skill; restart Codex to pick them up)

### Cursor / Windsurf / others (skills.sh installer)
```
npx skills add architectcms/architect-skills            # installs into detected agents
npx skills add architectcms/architect-skills -a cursor  # target one agent
```

### Anything else
Copy a `skills/<name>/` folder into your agent's skills directory (`.claude/skills/`, `.agents/skills/`, etc.).

## Skills
| Skill | Purpose |
| --- | --- |
| architect-setup | Install CLI + log in |
| architect-models | Define/evolve content models |
| architect-entries | Create/read/update entries |
| architect-types | Generate TypeScript types |
| architect-localization | Locale model + provider scaffold |
| architect-context | Context providers (segmentation/enrichment) |
| architect-context-actions | Run context actions |
| architect-lifecycle | Model lifecycle functions |
| architect-webhooks | Webhooks |
| architect-env | Environments |

## License
MIT
````

- [ ] **Step 2: Commit** (`docs: multi-platform install readme`).

### Task 2.2: CI validation of skills + manifests

**Files:** Create `.github/workflows/validate.yml`.

- [ ] **Step 1: Write `.github/workflows/validate.yml`** — fail the build if any `SKILL.md` lacks frontmatter `name`/`description`, or if the plugin/marketplace JSON is invalid:

```yaml
name: Validate
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - name: Validate SKILL.md frontmatter
        run: |
          node -e '
            const fs=require("fs"), path=require("path");
            const dir="skills"; let bad=0;
            for (const name of fs.readdirSync(dir)) {
              const f=path.join(dir,name,"SKILL.md");
              if(!fs.existsSync(f)){console.error("missing",f);bad++;continue;}
              const t=fs.readFileSync(f,"utf8");
              const m=t.match(/^---\n([\s\S]*?)\n---/);
              if(!m){console.error("no frontmatter",f);bad++;continue;}
              if(!/\nname:\s*\S/.test("\n"+m[1])){console.error("no name",f);bad++;}
              if(!/\ndescription:\s*\S/.test("\n"+m[1])){console.error("no description",f);bad++;}
            }
            process.exit(bad?1:0);
          '
      - name: Validate Claude plugin + marketplace JSON
        run: |
          node -e 'JSON.parse(require("fs").readFileSync(".claude-plugin/plugin.json","utf8"));JSON.parse(require("fs").readFileSync(".claude-plugin/marketplace.json","utf8"));console.log("json ok")'
```

- [ ] **Step 2: Commit + push; confirm the workflow is green** (`gh run list --limit 1`).

---

## Phase 3 — Publish & register

### Task 3.1: Tag a release and register on skills.sh

- [ ] **Step 1: Tag `v0.1.0`** so installers can pin:
```bash
git tag v0.1.0 && git push origin v0.1.0
```
- [ ] **Step 2: Verify skills.sh installability** (no submission form needed — GitHub *is* the registry; listing follows install telemetry):
```bash
npx skills add architectcms/architect-skills --all   # in a scratch dir; confirms discovery of skills/<name>/SKILL.md
```
Expected: the installer lists all `architect-*` skills and installs the selected ones. If it finds nothing, the layout is wrong (skills must be at `skills/<name>/SKILL.md`).
- [ ] **Step 3 (optional): npm publish** `@architectcms/skills` for an additional install channel (`npm publish` from the repo; `files` already scopes the tarball). Add a changeset-free manual publish or reuse the org's Trusted Publishing pattern.
- [ ] **Step 4: Commit** any doc tweaks (`chore: v0.1.0 release`).

---

## Phase 4 — CLI-dependent skills (gated on the companion plan)

> **Do not start until** `@architectcms/cli` with `contexts`, `context-actions`, `lifecycle`, `webhooks`, `env` is published (companion plan `architect-sdk/docs/plans/2026-06-12-management-expansion.md`). Bump the README's minimum CLI version to that release.

### Task 4.1: `architect-context` skill

**Files:** Create `skills/architect-context/SKILL.md`, `skills/architect-context/references/strategies.md`.

curl→CLI mapping: `/api/context-providers` GET → `architect contexts list`/`pull`; POST/PUT → `architect contexts push <file>`.

- [ ] **Step 1: Write `skills/architect-context/SKILL.md`:**

```markdown
---
name: architect-context
description: Create and manage Architect CMS context providers — audience segmentation and data enrichment that vary content by locale, customer tier, geography, etc. — via the architect CLI. Use when content should change based on who's viewing or request context.
---

# architect-context

Manage context providers with the `architect` CLI.

## Prerequisites
`architect-setup` done; the source model exists (`architect-models`) and is marked `isContextModel: true` with a `keyField`.

## List / pull
```bash
architect contexts list
architect contexts pull --out architect/contexts.json
```

## Create / update
Push a JSON array of providers (no `id` = create, `id` = update):
```json
[
  {
    "name": "Localization",
    "sourceModelId": "locale",
    "derivationPath": [],
    "hierarchyRelations": ["parent"]
  },
  {
    "name": "LoyaltyTier",
    "sourceModelId": "customer",
    "derivationPath": ["tier"],
    "hierarchyRelations": null
  }
]
```
```bash
architect contexts push architect/contexts.json
```

## Shape that matters
- `sourceModelId` — the context model to resolve against.
- `derivationPath` — `[]` means identity (use the source model's `keyField`); `["field"]` reads a field; `["relation","field"]` traverses a relation then reads a field.
- `hierarchyRelations` — relation field name(s) (e.g. `["parent"]`) enable most-specific-first inheritance; `null`/`[]` = flat. Resolution policy is not configurable (most-specific-first).

See `references/strategies.md` for segmentation patterns.

## Related skills
- `architect-models`, `architect-localization`, `architect-context-actions`
```

- [ ] **Step 2: Write `references/strategies.md`** — port segmentation strategies from `/Users/nkranendonk/Projects/architect/skills/context/references/strategies.md`, reframed as `contexts push` provider JSON.
- [ ] **Step 3: Commit** (`feat: architect-context skill`).

### Task 4.2: `architect-context-actions` skill

**Files:** Create `skills/architect-context-actions/SKILL.md`.

- [ ] **Step 1: Write `skills/architect-context-actions/SKILL.md`:**

```markdown
---
name: architect-context-actions
description: List and execute Architect CMS context actions — operations bound to a context provider (e.g. "translate to locale") that produce context-specific content. Use when the user wants to run a context-bound action like generating localized field values.
---

# architect-context-actions

## Prerequisites
`architect-setup` done; a context provider exists (`architect-context`).

## List
```bash
architect context-actions list
```

## Run
```bash
architect context-actions run <actionId> --param entryId=<id> --param locale=fr
```
`--param key=value` is repeatable; pass the params the action expects.

## Related skills
- `architect-context`, `architect-entries`
```

- [ ] **Step 2: Commit** (`feat: architect-context-actions skill`).

### Task 4.3: `architect-lifecycle` skill

**Files:** Create `skills/architect-lifecycle/SKILL.md`, `skills/architect-lifecycle/references/examples.md`.

- [ ] **Step 1: Write `skills/architect-lifecycle/SKILL.md`:**

```markdown
---
name: architect-lifecycle
description: Add and manage Architect CMS lifecycle functions — JavaScript handlers (beforeSave, afterSave, beforeDelete, afterDelete) that run server-side on a model's entries, e.g. to derive slugs, validate, or denormalize. Use when content needs computed fields or server-side rules.
---

# architect-lifecycle

Manage per-model lifecycle functions with the `architect` CLI.

## Prerequisites
`architect-setup` done; the target model exists.

## List
```bash
architect lifecycle list --model Article
```

## Add
Write the function body to a `.js` file, then:
```bash
architect lifecycle add --model Article --event beforeSave --code-file ./slug.js
```
`./slug.js` (runs server-side in a sandbox; `entry` is in scope):
```js
entry.data.slug = entry.data.title.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-|-$/g, "")
```
Events: `beforeSave`, `afterSave`, `beforeDelete`, `afterDelete`.

## Remove
```bash
architect lifecycle rm <functionId> --model Article
```

See `references/examples.md` for more handlers (validation, denormalization, timestamps).

## Related skills
- `architect-models`, `architect-entries`
```

- [ ] **Step 2: Write `references/examples.md`** — port from `/Users/nkranendonk/Projects/architect/skills/lifecycle/references/examples.md`, each example as a `--code-file` snippet.
- [ ] **Step 3: Commit** (`feat: architect-lifecycle skill`).

### Task 4.4: `architect-webhooks` skill

**Files:** Create `skills/architect-webhooks/SKILL.md`.

- [ ] **Step 1: Write `skills/architect-webhooks/SKILL.md`:**

```markdown
---
name: architect-webhooks
description: Manage Architect CMS webhooks — register endpoints that fire on content events (entry published/updated/deleted), send test deliveries, and remove them — via the architect CLI. Use when the user wants real-time notifications or to integrate external systems.
---

# architect-webhooks

## Prerequisites
`architect-setup` done.

## List
```bash
architect webhooks list
```

## Add
```bash
architect webhooks add --url https://example.com/hook --events entry.published,entry.deleted
```

## Test a delivery
```bash
architect webhooks test <webhookId>
```

## Remove
```bash
architect webhooks rm <webhookId>
```

## Related skills
- `architect-entries`, `architect-env`
```

- [ ] **Step 2: Commit** (`feat: architect-webhooks skill`).

### Task 4.5: `architect-env` skill

**Files:** Create `skills/architect-env/SKILL.md`.

- [ ] **Step 1: Write `skills/architect-env/SKILL.md`:**

```markdown
---
name: architect-env
description: Manage Architect CMS environments — list environments and create new ones with a promotion target (e.g. staging → production) — via the architect CLI. Use when the user wants environment isolation, a new environment, or to understand promotion flow.
---

# architect-env

## Prerequisites
`architect-setup` done.

## List
```bash
architect env list     # shows id, role, and promotesTo
```

## Create
```bash
architect env create --name "Staging" --promotes-to production
```

## Switching environments
The CLI operates against the environment you logged in with. To target another environment, re-run `architect login --environment-id <id>` (or set `ARCHITECT_ENV`).

## Related skills
- `architect-setup`, `architect-models`
```

- [ ] **Step 2: Commit** (`feat: architect-env skill`).

### Task 4.6: Update manifests + README for the new skills, validate, release

- [ ] **Step 1: Bump versions** in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` to `0.2.0`; bump `package.json` to `0.2.0`. (Skills auto-discover, so no per-skill manifest entry needed.)
- [ ] **Step 2: Update README** minimum CLI version to the published companion-plan CLI release.
- [ ] **Step 3: Validate** (`claude plugin validate .`) and run the CI validator locally (the node frontmatter check from Task 2.2).
- [ ] **Step 4: Live test** each new skill end-to-end against a real env via Claude Code (install the updated plugin, invoke `/architect-cms:architect-lifecycle`, etc., and confirm the `architect` commands it runs succeed).
- [ ] **Step 5: Tag `v0.2.0`, push, verify `npx skills add architectcms/architect-skills --all` lists all ten skills.** Commit (`chore: v0.2.0 — context/lifecycle/webhooks/env skills`).

---

## Out of scope (note for later)
- **MCP server retirement.** `@architect-cms/mcp` overlaps these CLI skills; decide whether to deprecate it or keep it for non-terminal hosts. Not part of this plan.
- **Migrating/deprecating `../architect/skills`.** This repo supersedes the curl-based skills; once shipped, point that package's README here and deprecate it (separate task).
- **Submitting to the official `openai/skills` catalog** (a PR to that repo) for Codex catalog discovery — optional once skills are stable.

---

## Self-Review notes (for the implementer)
- **Frontmatter is mandatory.** The source skills in `../architect/skills` have none; every ported `SKILL.md` here must start with `--- name / description ---`. The CI validator (Task 2.2) enforces it.
- **No secrets, ever.** Skills tell users to run `architect login`; examples use placeholder keys only. The repo is public (CLAUDE.md guardrails).
- **Phase ordering:** Phases 0–3 ship today against `@architectcms/cli@0.1.1`. Phase 4 is blocked on the companion CLI/SDK expansion plan — don't author those five skills until their `architect` commands exist, or they'll document commands that error.
- **Single-repo marketplace:** `marketplace.json` lists one plugin with `source: "."`; `plugin.json` + `marketplace.json` both live in root `.claude-plugin/`; skills at root `skills/`. Run `claude plugin validate .` after any manifest change.
- **Command names must match the CLI.** Every `architect …` invocation in a skill must match a real command/flag from the companion plan (`contexts`, `context-actions`, `lifecycle`, `webhooks`, `env` and their subcommands/flags). Cross-check against the CLI `--help` before publishing.
- **Naming consistency:** plugin name `architect-cms`, marketplace name `architect-skills`, package `@architectcms/skills`, skills `architect-<domain>` — keep these exact across manifests, README, and install commands.
