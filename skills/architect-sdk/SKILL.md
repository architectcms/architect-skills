---
name: architect-sdk
description: Set up the Architect CMS SDK (@architectcms/sdk) in an application — install it, scaffold a delivery and/or preview client with an example .env, then fetch content from the CMS and wire it into the app. Use when the user wants their app to read content from Architect.
---

# architect-sdk

Wire `@architectcms/sdk` into an app and pull CMS content into it. This is the inverse of `architect-extract` (which pushes hard-coded content up to the CMS).

## 1. Install

```bash
npm install @architectcms/sdk
```

## 2. Pick the client + key

| Client | Key prefix | Use |
| --- | --- | --- |
| `ArchitectDelivery` | `arch_delivery_…` | Published content (production) |
| `ArchitectPreview` | `arch_preview_…` | Drafts / unpublished (preview builds) |

`ArchitectManagement` (`arch_mgmt_…`) also exists but app code should prefer delivery/preview.

## 3. Example `.env`

Create `.env.example` (and ensure `.env` is git-ignored):

```
ARCHITECT_ORG_ID=org_…
ARCHITECT_ENV_ID=env_…
ARCHITECT_DELIVERY_KEY=arch_delivery_…
ARCHITECT_PREVIEW_KEY=arch_preview_…
```

```bash
grep -qxF '.env' .gitignore || echo '.env' >> .gitignore
```

## 4. Typed data-fetching layer

First generate types (see `architect-types`: `architect types generate`), then a small client module:

```ts
// lib/architect.ts
import { ArchitectDelivery, ArchitectPreview } from '@architectcms/sdk'

const base = {
  organizationId: process.env.ARCHITECT_ORG_ID!,
  environmentId: process.env.ARCHITECT_ENV_ID!,
  baseUrl: 'https://api.architectcms.com',
}

export const cms =
  process.env.ARCHITECT_PREVIEW === '1'
    ? new ArchitectPreview({ ...base, apiKey: process.env.ARCHITECT_PREVIEW_KEY! })
    : new ArchitectDelivery({ ...base, apiKey: process.env.ARCHITECT_DELIVERY_KEY! })

export const getArticles = () => cms.entries.model('article').fetch()
export const getArticle = (id: string) => cms.entries.get(id)
```

## 5. Verify the fetch

Before editing UI, confirm content comes back:

```bash
node --env-file=.env -e "import('@architectcms/sdk').then(async ({ArchitectDelivery})=>{const c=new ArchitectDelivery({organizationId:process.env.ARCHITECT_ORG_ID,environmentId:process.env.ARCHITECT_ENV_ID,apiKey:process.env.ARCHITECT_DELIVERY_KEY,baseUrl:'https://api.architectcms.com'});console.log((await c.entries.model('article').fetch()).length+' entries')})"
```

## 6. Hook it up (propose → confirm before editing components)

- **Default:** wire **one** representative spot end-to-end as a copyable pattern (detect the framework — Next.js server component, Remix loader, plain React fetch, etc. — and follow its convention). Show the diff and get confirmation before editing the user's files. The user extends the rest.
- **Escalation — replace ALL hard-coded content:** only when the user explicitly asks to replace all hard-coded content, find every hard-coded spot that maps to CMS content and convert it to SDK calls. Still **propose the full set of edits and confirm before applying.**

Use `withContext(...)` for context-aware content and `cms.assets.imageUrl(id, { width, format })` for images, per the SDK README.

## Related skills

- `architect-types` — generate types the data layer uses
- `architect-extract` — push hard-coded content up to the CMS (the inverse direction)
- `architect-setup` — where each kind of key lives
