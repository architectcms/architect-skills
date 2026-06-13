# Lifecycle Function Examples

Each example is a complete `--code-file` for `architect lifecycle add`. The suggested `--events`/`--timing` flags are noted above each.

Handler parameters:

- `entry` — `{ id, modelId, data }` (`id` is null on creates).
- `context` — `{ event, user, environment, isNew, previousData }`.
- `services` — `{ http: { get/post/put/delete }, entries: { find/get/create/update }, log, kv: { get/set } }`.

## Validation (`--events onCreate,onUpdate --timing before`)

Required fields:

```js
function handler(entry, context, services) {
  const required = ["name", "email"]
  const missing = required.filter((field) => !entry.data[field])
  if (missing.length > 0) {
    throw new Error(`Missing required fields: ${missing.join(", ")}`)
  }
  return { entry }
}
```

Price rules:

```js
function handler(entry, context, services) {
  const { price, salePrice } = entry.data
  if (price !== undefined && price < 0) {
    throw new Error("Price cannot be negative")
  }
  if (salePrice !== undefined && salePrice >= price) {
    throw new Error("Sale price must be less than regular price")
  }
  return { entry }
}
```

Unique constraint:

```js
async function handler(entry, context, services) {
  if (entry.data.email) {
    const existing = await services.entries.find({
      modelId: entry.modelId,
      filter: { email: entry.data.email.toLowerCase() },
    })
    if (existing.some((e) => e.id !== entry.id)) {
      throw new Error("An entry with this email already exists")
    }
  }
  return { entry }
}
```

## Transformation (`--events onCreate,onUpdate --timing before`)

Slug from title (accent-safe):

```js
function handler(entry, context, services) {
  if (entry.data.title && !entry.data.slug) {
    entry.data.slug = entry.data.title
      .toLowerCase()
      .normalize("NFD")
      .replace(/[\u0300-\u036f]/g, "")
      .replace(/[^a-z0-9]+/g, "-")
      .replace(/^-|-$/g, "")
      .substring(0, 100)
  }
  return { entry }
}
```

Computed total:

```js
function handler(entry, context, services) {
  const { quantity, unitPrice, discount = 0 } = entry.data
  if (quantity !== undefined && unitPrice !== undefined) {
    const subtotal = quantity * unitPrice
    entry.data.total = Math.round(subtotal * (1 - discount / 100) * 100) / 100
  }
  return { entry }
}
```

Timestamps and author tracking:

```js
function handler(entry, context, services) {
  const now = new Date().toISOString()
  if (context.isNew) {
    entry.data.createdAt = now
  }
  entry.data.updatedAt = now
  if (context.user) {
    entry.data.lastModifiedBy = context.user.id
  }
  return { entry }
}
```

## Side effects (`--events onCreate,onUpdate --timing after`)

Notify an external system (don't fail the save if the sync fails):

```js
async function handler(entry, context, services) {
  try {
    await services.http.post("https://api.example.com/sync", {
      id: entry.id,
      type: entry.modelId,
      data: entry.data,
    })
  } catch (error) {
    services.log("Sync failed: " + error.message)
  }
  return { success: true }
}
```

Denormalize onto related entries (e.g. category rename → products):

```js
async function handler(entry, context, services) {
  if (!context.isNew && context.previousData.name !== entry.data.name) {
    const products = await services.entries.find({
      modelId: "product",
      filter: { category: entry.id },
    })
    for (const product of products) {
      await services.entries.update(product.id, { data: { categoryName: entry.data.name } })
    }
    services.log(`Updated ${products.length} products`)
  }
  return { success: true }
}
```

## Delete guard (`--events onDelete --timing after`)

> `onDelete` only supports `after` timing — use it for cleanup and notifications, not blocking.

```js
async function handler(entry, context, services) {
  services.log(`Entry ${entry.id} deleted by ${context.user ? context.user.id : "unknown"}`)
  await services.http.post("https://api.example.com/deleted", { id: entry.id, model: entry.modelId })
  return { success: true }
}
```

## Tips

- One concern per function — compose several small handlers rather than one big one.
- Guard against null/undefined before reading `entry.data` fields.
- External HTTP calls are slow; keep `before` handlers fast so saves stay snappy.
- Use `services.log()` to debug; logs surface in the web app.
