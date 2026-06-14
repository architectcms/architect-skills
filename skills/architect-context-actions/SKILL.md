---
name: architect-context-actions
description: List and execute Architect CMS context actions — operations bound to a context provider (e.g. "translate to locale") that produce context-specific content for an entry. Use when the user wants to run a context-bound action like generating localized field values.
---

# architect-context-actions

Run context actions with the `architect` CLI.

## Prerequisites

`architect-setup` done; a context provider exists (`architect-context`).

## List

Actions are listed per provider:

```bash
architect context-actions list --provider <providerId>
```

Find provider ids with `architect contexts list`.

## Run

Execute an action against an entry, under a specific context value:

```bash
architect context-actions run <actionId> --entry <entryId> --context-value fr-FR --model <modelId>
```

- `--entry` — the entry the action operates on.
- `--context-value` — the context to execute under (e.g. a locale code).
- `--model` — the entry's model.

Example: a "Translate" action bound to the Localization provider, run with `--context-value fr-FR`, writes French values into the entry's `fr-FR` context overrides.

## Workflow: localize an entry

1. `architect contexts list` → find the Localization provider id.
2. `architect context-actions list --provider localization` → find the translate action id.
3. `architect context-actions run <actionId> --entry <entryId> --context-value fr-FR --model article`
4. Confirm: `architect entries pull --model article` → the entry's `contextOverrides` now has `fr-FR` values.

## Related skills

- `architect-context`, `architect-entries`
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
