# Context Segmentation Strategies

Patterns for context providers, each shown as the model + provider JSON you `architect models push` / `architect contexts push`.

## Identity (key field)

The simplest provider: the context value *is* the source entry's key field. Used for locales, brands, sites.

Model:

```json
{
  "name": "Locale",
  "isContextModel": true,
  "keyField": "code",
  "fields": [
    { "name": "code", "type": "text", "required": true },
    { "name": "name", "type": "text", "required": true }
  ]
}
```

Provider:

```json
{ "name": "Localization", "sourceModel": "locale", "derivationPath": [], "hierarchyRelations": null }
```

Resolution: the consumer passes `code=fr-FR`; entries with a `fr-FR` override return that value.

## Field derivation

Read the context value from a field on the source entry rather than its key.

```json
{ "name": "LoyaltyTier", "sourceModel": "customer", "derivationPath": ["tier"], "hierarchyRelations": null }
```

Resolution: `customer.tier` → `"gold"`. Use for: customer tiers, account status, membership type.

## Relation traversal

Follow a relation, then read a field on the target.

Customer → loyaltyProgram (relation) → tier (field):

```json
{ "name": "LoyaltyTier", "sourceModel": "customer", "derivationPath": ["loyaltyProgram", "tier"], "hierarchyRelations": null }
```

Use for: tier via program membership, region via store, category via product line — any case where the segmenting value lives one hop away.

## Hierarchy with fallback

Self-referencing relation(s) on the source model give values a parent chain; resolution is most-specific-first up the chain.

Model (note the self-referencing `parent`):

```json
{
  "name": "Locale",
  "isContextModel": true,
  "keyField": "code",
  "fields": [
    { "name": "code", "type": "text", "required": true },
    { "name": "name", "type": "text", "required": true },
    { "name": "parent", "type": "model", "targetModelIds": ["locale"], "multiple": false }
  ]
}
```

Provider:

```json
{ "name": "Localization", "sourceModel": "locale", "derivationPath": [], "hierarchyRelations": ["parent"] }
```

Resolution for `fr-FR`: try `fr-FR` → fall back to `fr` (its parent) → fall back to the un-contextualized value. The same pattern fits region trees (city → state → country) and category trees.

`architect contexts pull` shows the resulting tree in `possibleValues` (children nested under parents) — use it to confirm the hierarchy wired up correctly.

## Choosing a pattern

| Need | Pattern |
| --- | --- |
| Locale / brand / site switching | Identity, usually + hierarchy |
| Customer tiers, plan levels | Field derivation |
| Segment by something one relation away | Relation traversal |
| Regional or categorical fallback | Hierarchy |

Tips:

- One provider per axis of variation. Locale and loyalty tier are two providers, not one.
- Keep context models small and stable — they're configuration, not content.
- After pushing a provider, mark the fields that should vary by it (`"contexts": ["<providerId>"]` on the field — see `architect-models`).
