# Drafts and migration

| Draft | `$schema` URI |
|---|---|
| 2020-12 | `https://json-schema.org/draft/2020-12/schema` |
| 2019-09 | `https://json-schema.org/draft/2019-09/schema` |
| draft-07 | `http://json-schema.org/draft-07/schema#` |
| draft-06 | `http://json-schema.org/draft-06/schema#` |
| draft-04 | `http://json-schema.org/draft-04/schema#` |

Note the scheme: older drafts use `http://` with a trailing `#`, while 2019-09+ use `https://` with no `#`. A wrong URI makes validators fall back to a default dialect, or fail.

## draft-04 → draft-06/07

- `id` → `$id`.
- `exclusiveMinimum: true, minimum: 5` → `exclusiveMinimum: 5` (now a number, and `minimum` is removed).
- New in draft-06: `const`, `contains`, `propertyNames`, `examples`, and boolean schemas.
- New in draft-07: `if`/`then`/`else`, `readOnly`, `writeOnly`, `$comment`, and `contentMediaType`/`contentEncoding`.

## draft-07 → 2019-09

| draft-07 | 2019-09 |
|---|---|
| `definitions` | `$defs` (also update every `#/definitions/...` ref) |
| `dependencies: {"a": ["b"]}` | `dependentRequired: {"a": ["b"]}` |
| `dependencies: {"a": {schema}}` | `dependentSchemas: {"a": {schema}}` |
| `$ref` with siblings (siblings ignored) | Siblings are now evaluated. **Behavior change**: check that old ignored siblings don't start rejecting data. Ajv already applied siblings in draft-07, so users on Ajv see no change, but users on spec-following validators (Python `jsonschema` and others) do. |
| `id`-style fragments `"$id": "#foo"` | `"$anchor": "foo"` |
| — | New: `unevaluatedProperties`, `unevaluatedItems`, `minContains`/`maxContains`, `$recursiveRef`/`$recursiveAnchor`, `$vocabulary`, `contentSchema`, `deprecated` |
| `format` asserted by many validators | `format` is an annotation by default |

## 2019-09 → 2020-12

| 2019-09 | 2020-12 |
|---|---|
| `items: [a, b]` (tuple) | `prefixItems: [a, b]` |
| `additionalItems: s` (with tuple `items`) | `items: s` |
| `items: s` (all items) | `items: s` (unchanged) |
| `$recursiveAnchor: true` / `$recursiveRef: "#"` | `$dynamicAnchor: "name"` / `$dynamicRef: "#name"` |
| `"$id": "foo#"` (empty fragment allowed) | No fragment in `$id` |
| `unevaluatedItems` sees `items`/`additionalItems` | Sees `prefixItems`/`items`/`contains` |
| format vocabulary | Split into `format-annotation` (default) and `format-assertion` |

Also new in 2020-12: an official bundling process (embedded resources with `$id` under `$defs`).

## Migration procedure

1. Update `$schema`.
2. Apply the keyword mapping for each draft hop in order (04 → 07 → 2019-09 → 2020-12).
3. Fix every `$ref` path that pointed into `definitions` or used a renamed keyword.
4. Before editing, record the old results with a spec-accurate validator (Python `jsonschema` handles draft-07 `$ref` siblings correctly, while Ajv doesn't). Compile the new schema with Ajv's `strictSchema: true` to catch leftover keywords, then validate existing sample and fixture data with both the old and new schemas. Any difference in results is a behavior change you need to explain or fix (the `$ref`-sibling change and `format` assertion are the usual culprits).

## OpenAPI 3.0 schema objects → JSON Schema 2020-12 / OpenAPI 3.1

| OpenAPI 3.0 | 2020-12 / OAS 3.1 |
|---|---|
| `nullable: true` | add `"null"` to `type` (`"type": ["string", "null"]`). With `enum`, also add `null` to the enum. With `$ref`, use `anyOf: [{$ref}, {type: "null"}]`. |
| `exclusiveMinimum: true` + `minimum: n` | `exclusiveMinimum: n` |
| `example: x` | `examples: [x]` |
| `type` must be a single string | type arrays allowed |
| `$ref` siblings ignored | siblings allowed and applied |
| no `const`, `if/then/else`, `prefixItems`, `$defs`, `unevaluated*`, `patternProperties`... | all available |
| `format: binary` / `byte` for files | `contentMediaType` / `contentEncoding: base64` |

`discriminator`, `xml`, `externalDocs`, and `example` (deprecated) are OAS-dialect keywords. Plain Ajv strict mode rejects them, so use `strict: false`, or `discriminator: true` plus `ajv.addKeyword(...)`.
