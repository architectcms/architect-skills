---
name: architect-lifecycle
description: Add and manage Architect CMS lifecycle functions — JavaScript handlers that run server-side before or after an entry is created, updated, or deleted, e.g. to derive slugs, validate, or denormalize. Use when content needs computed fields or server-side rules.
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

Write the handler to a `.js` file, then:

```bash
architect lifecycle add --model Article --name slugify --events onCreate,onUpdate --timing before --code-file ./slug.js
```

- `--events` — comma-separated: `onCreate`, `onUpdate`, `onDelete`.
- `--timing` — `before` or `after` the event (default `after`; `onDelete` supports `after` only).
- `--name` — a name for the function.

The code file **must** define `function handler(entry, context, services)` (the CLI rejects anything else). `./slug.js`:

```js
function handler(entry, context, services) {
  if (entry.data.title && !entry.data.slug) {
    entry.data.slug = entry.data.title
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, "-")
      .replace(/^-|-$/g, "")
  }
  return { entry }
}
```

Handler contract:

- Runs server-side in a sandbox (no filesystem; network via `services.http` only).
- `before` handlers: mutate `entry.data` and `return { entry }` to transform, or `throw new Error("…")` to block the operation.
- `after` handlers: side effects only — `return { success: true }`.
- `async function handler(…)` is allowed; use `await` with `services.*`.

## Remove

```bash
architect lifecycle rm <functionId>
```

Get the id from `lifecycle list`.

See `references/examples.md` for more handlers (validation, denormalization, timestamps, delete guards), and `references/execution-model.md` for the sandbox runtime: the `services` object (`http`/`entries`/`log`/`kv`), execution limits, retry/idempotency semantics, and error types.

## Related skills

- `architect-models`, `architect-entries`
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
