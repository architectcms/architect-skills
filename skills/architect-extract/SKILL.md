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
   ```bash
   architect assets upload ./path/to/image.png --alt "…"
   ```
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
