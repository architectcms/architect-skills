# Architect CMS — Agent Skills

CLI-driven [Agent Skills](https://agentskills.io) that let Claude, Codex, Cursor, and other terminal agents manage [Architect CMS](https://architectcms.com): model content, manage entries, generate types, set up localization, context providers, lifecycle functions, webhooks, and environments — all through the `architect` CLI.

> **Requires** the CLI: `npm i -g @architectcms/cli` (>= 0.2.0 for the context/lifecycle/webhooks/env skills). Run `architect login` once. See the `architect-setup` skill.

## Install

### Claude Code (plugin marketplace)

```
/plugin marketplace add architectcms/architect-skills
/plugin install architect-cms@architect-skills
```

### Codex (plugin marketplace)

```
codex plugin marketplace add architectcms/architect-skills
```

Then install the `architect-cms` plugin from `/plugins` (or `codex plugin marketplace list`). One install pulls in all the skills; restart Codex to pick them up.

### Cursor / Windsurf / others (skills.sh installer)

```
npx skills add architectcms/architect-skills            # installs into detected agents
npx skills add architectcms/architect-skills -a cursor  # target one agent
```

### Anything else

Copy a `skills/<name>/` folder into your agent's skills directory (`.claude/skills/`, `.agents/skills/`, etc.).

## Skills

| Skill | Purpose |
| --- | --- |
| architect-setup | Install CLI + log in |
| architect-models | Define/evolve content models |
| architect-entries | Create/read/update entries |
| architect-types | Generate TypeScript types |
| architect-localization | Locale model + provider scaffold |
| architect-context | Context providers (segmentation/enrichment) |
| architect-context-actions | Run context actions |
| architect-lifecycle | Model lifecycle functions |
| architect-webhooks | Webhooks |
| architect-env | Environments |
| architect-sdk | Install the SDK, fetch content + wire it into an app |
| architect-mcp | Install/configure the Architect MCP server for content access |
| architect-extract | Migrate hard-coded app content into the CMS |

## License

MIT
