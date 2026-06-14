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

This fetches every model in the current environment and writes one `export interface` per model (fields typed from the schema; relations typed by target). `--output` defaults to `./architect-types.ts`.

## Workflow

1. Change schema (`architect-models`).
2. Regenerate: `architect types generate --output src/architect-types.ts`.
3. Import the interfaces in app code.

## Related skills

- `architect-models`
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
