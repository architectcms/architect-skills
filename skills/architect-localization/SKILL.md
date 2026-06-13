---
name: architect-localization
description: Set up localization in Architect CMS — a Locale context model, seeded locale entries, and a Localization context provider with language→region fallback — via `architect init`. Use when the user wants multi-locale/translated content.
---

# architect-localization

Scaffold localization with one command via `architect init`.

## Prerequisites

`architect-setup` done.

## Interactive

```bash
architect init        # answer the localization prompts (locales, default, hierarchy)
```

## From a config (repeatable / CI)

```bash
architect init --config init.json --yes
```

Example `init.json` (full version with a starter model in `examples/init.fr-en.json`):

```json
{
  "localization": {
    "enabled": true,
    "modelName": "Locale",
    "providerName": "Localization",
    "locales": [
      { "code": "en-US", "name": "English (US)" },
      { "code": "fr-FR", "name": "French (France)" }
    ],
    "defaultLocale": "en",
    "hierarchy": true
  }
}
```

With `hierarchy: true`, a region like `fr-FR` falls back to its base language `fr` (created automatically), then to the default. Resolution is most-specific-first.

## What it creates

- A `Locale` context model (key field `code`).
- One locale entry per code (base languages first, regions linked to them via a `parent` relation).
- A `Localization` context provider (`derivationPath: []`, `hierarchyRelations: ["parent"]`).

Verify afterwards:

```bash
architect contexts list             # shows the Localization provider
architect entries pull --model Locale
```

## Related skills

- `architect-context` (general context providers), `architect-models`
