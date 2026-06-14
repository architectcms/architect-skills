---
name: architect-context
description: Create and manage Architect CMS context providers — audience segmentation and data enrichment that vary content by locale, customer tier, geography, etc. — via the architect CLI. Use when content should change based on who's viewing or request context.
---

# architect-context

Manage context providers with the `architect` CLI.

## Prerequisites

`architect-setup` done; the source model exists (`architect-models`) and is marked `isContextModel: true` with a `keyField`.

## List / inspect / pull

```bash
architect contexts list
architect contexts get <id>
architect contexts pull --out architect/contexts.json
```

Pulled providers include `possibleValues` — the resolved value tree (with hierarchy children) the provider currently offers.

## Create / update

Push a JSON array of providers (no `id` = create, `id` = update):

```json
[
  {
    "name": "Localization",
    "sourceModel": "locale",
    "derivationPath": [],
    "hierarchyRelations": ["parent"]
  },
  {
    "name": "LoyaltyTier",
    "sourceModel": "customer",
    "derivationPath": ["tier"],
    "hierarchyRelations": null
  }
]
```

```bash
architect contexts push architect/contexts.json
```

## Shape that matters

- `sourceModel` — the context model (by id) to resolve against.
- `derivationPath` — `[]` means identity (use the source model's `keyField`); `["field"]` reads a field; `["relation", "field"]` traverses a relation then reads a field.
- `hierarchyRelations` — relation field name(s) (e.g. `["parent"]`) enable most-specific-first inheritance; `null`/`[]` = flat. Resolution policy is not configurable (most-specific-first).
- `queryParam` — the key consumers pass to select a context value (defaults to the source model's `keyField`).

See `references/strategies.md` for segmentation patterns.

## How fields opt in

A model field varies by context when its `contexts` array names the provider (see `architect-models`). Entries then store per-context values in `contextOverrides`.

## Related skills

- `architect-models`, `architect-localization`, `architect-context-actions`
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
