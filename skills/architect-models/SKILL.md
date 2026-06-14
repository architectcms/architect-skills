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
architect models pull                                # table to stdout; add --json for raw JSON
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
{
  "name": "Locale",
  "isContextModel": true,
  "keyField": "code",
  "fields": [
    { "name": "code", "type": "text", "required": true },
    { "name": "name", "type": "text", "required": true },
    { "name": "parent", "type": "model", "targetModelIds": ["locale"], "multiple": false }
  ]
}
```

## Field types

See `references/field-types.md` for every field type (`text`, `richtext`, `number`, `boolean`, `date`, `model` (relations), `select`, `key`, `json`, `array`, …) and their options.

## Workflows

- **Add a field:** `architect models pull --out m.json` → add the field object to the model's `fields` → `architect models push m.json`.
- **Generate types after a schema change:** run the `architect-types` skill.

## Tips

- Field names are camelCase (`productName`, not `product_name`).
- `targetModelIds` is always an array, even for a single target.
- Add `description` to models and fields — it doubles as editor help text.

## Related skills

- `architect-entries`, `architect-context`, `architect-types`
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
