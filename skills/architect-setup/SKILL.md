---
name: architect-setup
description: Set up the Architect CMS CLI — install @architectcms/cli, log in with a management API key, and confirm the connection. Use this first, before any other architect skill, or whenever architect commands fail with "Not logged in".
---

# architect-setup

Get a terminal agent ready to manage Architect CMS via the `architect` CLI.

## Overview

All other architect skills drive the `architect` CLI. This skill installs it and authenticates. Auth uses a **management API key** (`arch_mgmt_…`) created in the Architect web app — the CLI stores it at `~/.architect/credentials.json` (0600). Never read, print, or commit that file.

## Install the CLI

```bash
npm install -g @architectcms/cli   # provides the `architect` command
architect --version
```

## Log in

Interactive (prompts for key, org id, env id):

```bash
architect login
```

Non-interactive / CI (pass everything as flags — never paste a key into a file the agent commits):

```bash
architect login --api-key "$ARCHITECT_API_KEY" --organization-id "$ARCHITECT_ORG" --environment-id "$ARCHITECT_ENV"
```

Where to get these: create a **management** API key in the web app under Settings → API Keys; the org id and environment id are shown alongside it.

## Confirm

```bash
architect whoami
```

Expected output looks like:

```
org=org_xxxxxxxx env=development baseUrl=https://api.architectcms.com (key valid)
```

If `whoami` says "Not logged in", run `architect login` again.

## Tips

- Self-hosted? Pass `--base-url https://your-host` to `login` (default is `https://api.architectcms.com`).
- `architect logout` clears stored credentials.
- Add `--json` before any command (`architect --json models pull`) for machine-readable output.

## Related skills

- `architect-models` — define content models
- `architect-entries` — create/update content
