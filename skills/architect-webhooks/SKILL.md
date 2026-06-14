---
name: architect-webhooks
description: Manage Architect CMS webhooks — register endpoints that fire on content events (entry published/updated/deleted), send test deliveries, and remove them — via the architect CLI. Use when the user wants real-time notifications or to integrate external systems.
---

# architect-webhooks

Manage webhooks with the `architect` CLI.

## Prerequisites

`architect-setup` done.

## List

```bash
architect webhooks list     # id, name, url, enabled
```

## Add

```bash
architect webhooks add --name "Publish notifications" --url https://example.com/hook --events entry.published,entry.deleted
```

- `--events` — comma-separated, in `object.action` form (e.g. `entry.published`, `entry.updated`, `entry.deleted`, `model.updated`).
- `--url` — the HTTPS endpoint that receives the JSON delivery.

## Test a delivery

Send a test payload to verify the endpoint before relying on it:

```bash
architect webhooks test <webhookId>
```

## Remove

```bash
architect webhooks rm <webhookId>
```

## Notes

- Webhooks are asynchronous — they don't block the content operation. For in-band validation or transformation, use `architect-lifecycle` instead.
- Webhooks are environment-specific: they fire for events in the environment you're logged into.

## Related skills

- `architect-entries`, `architect-lifecycle`, `architect-env`
- Prefer the CLI. If the Architect MCP server is installed (see `architect-mcp`), the equivalent tools are in `architect-mcp/references/mcp-equivalents.md`.
