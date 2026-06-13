# architect-skills

Public repo of CLI-driven Agent Skills for [Architect CMS](https://architectcms.com). Each skill teaches a terminal agent (Claude Code, Codex, Cursor, …) to manage content via the `architect` CLI (`@architectcms/cli`). Distributed as a Claude plugin marketplace, Codex skills, and via `npx skills`.

## Repository hygiene — READ FIRST
This repository is PUBLIC; its full git history is visible. Never commit:
- Secrets — API keys (`arch_*`), tokens, passwords, `.env`, `~/.architect/credentials.json`. Skills must tell users to run `architect login`, never embed a key. Use placeholders (`arch_mgmt_…`) in examples.
- Internal planning/design docs beyond `docs/plans/`, or anything describing private backend internals.
- Local/agent artifacts (`.playwright-mcp/`, scratch files).

## Conventions
- Every skill lives at `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`).
- Skills are CLI-driven: they instruct the agent to run `architect …`. No raw HTTP/curl.
- Conventional Commits. Keep `marketplace.json`/`plugin.json` versions in sync with releases.
