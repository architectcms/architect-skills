---
name: architect-env
description: Manage Architect CMS environments — list environments and create new ones with a promotion target (e.g. staging → production) — via the architect CLI. Use when the user wants environment isolation, a new environment, or to understand promotion flow.
---

# architect-env

Manage environments with the `architect` CLI.

## Prerequisites

`architect-setup` done.

## List

```bash
architect env list     # id, displayName, role, promotesTo
```

## Create

```bash
architect env create --name "Staging" --promotes-to <environmentId>
```

`--promotes-to` takes the id of the environment this one promotes content into (e.g. a new `staging` promotes to `production`). Get ids from `env list`.

## Switching environments

The CLI operates against the environment you logged in with (`architect whoami` shows it). To target another environment, re-run:

```bash
architect login --environment-id <id>
```

## Workflow: dev → staging → production content

1. `architect env list` to see the promotion chain.
2. Author content in the development environment (`architect-entries`).
3. To copy content across environments with the CLI: `architect entries pull --model X --out X.json`, re-login with the target `--environment-id`, then `architect entries push --model X X.json`.

## Related skills

- `architect-setup`, `architect-models`
