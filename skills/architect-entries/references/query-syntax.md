# Query Syntax Reference

`architect entries pull` fetches a model's entries from the Architect content API. These notes describe how that API filters, sorts, and resolves relations server-side — useful context for understanding what pull returns and for application code querying the same API via the SDK.

## Filtering

Filters are exact-match by default and combine with AND logic. Comparison operators are appended to the field name with an underscore:

| Operator      | Description           | Example                       |
| ------------- | --------------------- | ----------------------------- |
| `_eq`         | Equals (default)      | `price_eq=29.99`              |
| `_ne`         | Not equals            | `status_ne=draft`             |
| `_lt`         | Less than             | `price_lt=50`                 |
| `_lte`        | Less than or equal    | `price_lte=50`                |
| `_gt`         | Greater than          | `price_gt=10`                 |
| `_gte`        | Greater than or equal | `price_gte=10`                |
| `_in`         | In list               | `status_in=draft,review`      |
| `_nin`        | Not in list           | `status_nin=archived`         |
| `_contains`   | Contains substring    | `name_contains=pro`           |
| `_startsWith` | Starts with           | `sku_startsWith=WGT`          |
| `_endsWith`   | Ends with             | `email_endsWith=@company.com` |
| `_null`       | Is null               | `deletedAt_null=true`         |
| `_notNull`    | Is not null           | `publishedAt_notNull=true`    |

Relation fields filter by entry id (`author=entry_author_123`, `categories_in=cat_1,cat_2`).

## Sorting

Single or multiple fields, `-` prefix for descending: `sort=status,-publishedAt`. `createdAt`, `updatedAt`, and `id` are always indexed.

## Pagination

`limit` + `offset`; responses include `total`, `hasMore`. Page N (0-indexed) = `offset = N * limit`.

## Relation resolution

Relations are **not** resolved by default — `data` holds plain entry ids. The API resolves them on request (`resolve=author,categories`, nested via `resolve=author.avatar`). Resolved targets arrive as full `{ id, data }` objects.

In `entries pull` output, server-resolved relations appear under each entry's `resolvedRelations` key while `data` keeps the raw ids — so pushing pulled JSON back never corrupts relation values.

## Best practices

1. Paginate large result sets — don't fetch everything at once.
2. Resolve relations explicitly when you need target data.
3. Filter on indexed fields (`createdAt`, `updatedAt`, `id`) when possible.
4. Multiple filters narrow results (AND logic).
