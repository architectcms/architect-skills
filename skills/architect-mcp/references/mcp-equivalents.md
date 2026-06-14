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
