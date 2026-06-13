# Field Types Reference

Every field type supported by Architect CMS, shown as the JSON field object you put in a model's `fields` array (then `architect models push <file>`).

Canonical `type` values: `text, richtext, textarea, email, url, string, model, relation, file, image, number, boolean, date, json, array, key, select`.

## Text types

### `text`

Short text. The workhorse for titles, names, codes.

```json
{ "name": "title", "type": "text", "required": true, "validation": { "minLength": 1, "maxLength": 200 } }
```

### `string`

Alias-style short text; supports an `options` list to render as a dropdown:

```json
{ "name": "status", "type": "string", "options": ["draft", "published", "archived"], "defaultValue": "draft" }
```

### `textarea`

Long plain text rendered as a multi-line editor.

```json
{ "name": "description", "type": "textarea", "validation": { "maxLength": 5000 } }
```

### `richtext`

HTML content with formatting (headings, bold, links, lists, images, code blocks).

```json
{ "name": "content", "type": "richtext" }
```

### `email`

Validated email address format.

```json
{ "name": "contactEmail", "type": "email", "required": true }
```

### `url`

Validated URL format.

```json
{ "name": "website", "type": "url" }
```

## Numbers, booleans, dates

### `number`

Integer or decimal values.

```json
{ "name": "price", "type": "number", "validation": { "min": 0, "max": 99999.99 }, "defaultValue": 0 }
```

### `boolean`

True/false, rendered as a toggle.

```json
{ "name": "featured", "type": "boolean", "defaultValue": false }
```

### `date`

Date value, `YYYY-MM-DD`.

```json
{ "name": "publishedAt", "type": "date" }
```

## Relations

### `model`

Link to entries of another model (also referred to as `relation`). `targetModelIds` is always an array, even for a single target; `multiple` controls whether the field holds one id or an array of ids.

```json
{ "name": "author", "type": "model", "targetModelIds": ["author"], "multiple": false, "required": true }
```

Multiple targets allowed:

```json
{ "name": "tags", "type": "model", "targetModelIds": ["tag"], "multiple": true }
```

Self-referential (hierarchies — categories, locales, regions):

```json
{ "name": "parent", "type": "model", "targetModelIds": ["category"], "multiple": false }
```

## Files and media

### `file` / `image`

Reference to an uploaded asset. `image` is the image-specific variant.

```json
{ "name": "featuredImage", "type": "image" }
```

```json
{ "name": "attachments", "type": "file", "multiple": true }
```

## Structured data

### `select`

Pick one value from a fixed list.

```json
{ "name": "tier", "type": "select", "options": ["bronze", "silver", "gold"], "defaultValue": "bronze" }
```

### `key`

Unique key/identifier field — used as a model's `keyField` (e.g. a locale code on a context model).

```json
{ "name": "code", "type": "key", "required": true }
```

### `json`

Arbitrary JSON data for flexible/dynamic content.

```json
{ "name": "metadata", "type": "json", "defaultValue": {} }
```

### `array`

Ordered list of values.

```json
{ "name": "highlights", "type": "array" }
```

## Common field properties

| Property       | Type    | Description                              |
| -------------- | ------- | ---------------------------------------- |
| `name`         | string  | Field identifier (camelCase)             |
| `type`         | string  | One of the canonical types above         |
| `displayName`  | string  | Human-readable label                     |
| `description`  | string  | Help text for editors                    |
| `required`     | boolean | Must have a value                        |
| `unique`       | boolean | No duplicate values allowed              |
| `defaultValue` | any     | Default when not provided                |
| `validation`   | object  | Type-specific rules (see below)          |
| `options`      | array   | Allowed values (`string`/`select`)       |
| `multiple`     | boolean | Field holds an array (`model`/`file`)    |
| `groupName`    | string  | Editor layout group the field belongs to |
| `contexts`     | array   | Context provider ids this field varies by |

## Validation rules by type

Text (`text`/`string`/`textarea`/`email`/`url`): `minLength`, `maxLength`, `pattern` (regex).

Number: `min`, `max`, `integer`.

Date: `minDate`, `maxDate`, `futureOnly`, `pastOnly`.

## Common patterns

Slug:

```json
{ "name": "slug", "type": "text", "required": true, "unique": true, "validation": { "pattern": "^[a-z0-9-]+$" } }
```

Price with currency:

```json
[
  { "name": "price", "type": "number", "validation": { "min": 0 } },
  { "name": "currency", "type": "select", "options": ["USD", "EUR", "GBP"], "defaultValue": "USD" }
]
```

Status with timestamps:

```json
[
  { "name": "status", "type": "select", "options": ["draft", "review", "published", "archived"], "defaultValue": "draft" },
  { "name": "publishedAt", "type": "date" }
]
```
