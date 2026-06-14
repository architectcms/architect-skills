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
architect entries pull --model Article          # table; add --json for raw JSON
```

Each pulled entry looks like:

```json
{
  "id": "42",
  "modelId": "article",
  "data": { "title": "Hello", "body": "<p>…</p>" },
  "version": 1,
  "createdAt": "2026-06-01T00:00:00.000Z",
  "updatedAt": "2026-06-01T00:00:00.000Z"
}
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

When pulling, related entries the server has resolved appear under `resolvedRelations` alongside `data` — read from there for display, but write plain ids into `data`.

## Workflows

- **Bulk import:** build the JSON array (no `id`s) → `architect entries push --model X file.json`.
- **Export/migrate:** `architect entries pull --model X --out X.json` from one env, `architect entries push` into another (re-login with the target env first).
- **Find and update:** pull, filter/edit the JSON locally (e.g. with `jq`), push the changed items back (keep their `id`s).

## Notes

- `--model` accepts the model name or id; the CLI resolves it.
- See `references/query-syntax.md` for how server-side filtering and relation resolution work on the underlying API.

## Related skills

- `architect-models`, `architect-context` (locale/segment-aware content)
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
