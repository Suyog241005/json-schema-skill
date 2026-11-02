# JSON Schema keywords (2020-12, with 2019-09 notes)

## Contents
1. Core (identity, references, structure)
2. Applicators (combining and conditional logic)
3. Object keywords
4. Array keywords
5. Validation (any type, numbers, strings)
6. Unevaluated
7. Format
8. Content
9. Meta-data (annotations)
10. Boolean schemas and common recipes

A schema is either an object or a boolean. `true` (or `{}`) accepts everything, and `false` (or `{"not": {}}`) rejects everything. Keywords that don't apply to the instance's type are ignored: `minLength` on a number passes. That's why `type` is usually needed alongside constraints.

## 1. Core

| Keyword | Meaning / gotchas |
|---|---|
| `$schema` | Dialect URI. Use it only at the root, or at the root of an embedded resource with `$id`. 2020-12: `https://json-schema.org/draft/2020-12/schema`. 2019-09: `https://json-schema.org/draft/2019-09/schema`. |
| `$id` | Canonical URI of this schema resource, and the base for resolving relative `$ref`s inside it. Must not contain a fragment, except an empty one in 2019-09. Use an absolute URL for published schemas. |
| `$ref` | Applies the referenced schema. In 2019-09+ it's a normal applicator, so siblings are also evaluated (in draft-07 and earlier, siblings of `$ref` are ignored). The value is a URI reference: `#/$defs/name`, `other.json`, `other.json#/$defs/x`, or `#anchor`. |
| `$defs` | Container for reusable subschemas. Replaces `definitions`, which 2020-12's meta-schema still allows for compatibility. Prefer `$defs`. |
| `$anchor` | Plain-name fragment (`"$anchor": "addr"` → `$ref: "#addr"`). Must match `^[A-Za-z_][-A-Za-z0-9._]*$`. |
| `$dynamicAnchor` / `$dynamicRef` | **2020-12.** Extensible recursion: `$dynamicRef: "#meta"` resolves to the *outermost* schema in the dynamic scope that declares `$dynamicAnchor: "meta"`. Used for extending recursive schemas, such as trees whose node type can be specialized. |
| `$recursiveAnchor` / `$recursiveRef` | **2019-09 only.** Predecessor of the above: `$recursiveAnchor: true` plus `$recursiveRef: "#"`. |
| `$vocabulary` | Only in meta-schemas. Declares which vocabularies the dialect uses. |
| `$comment` | Note for schema maintainers, with no effect on validation. Don't put user-facing text here; that goes in `description`. |

**JSON Pointer in `$ref`**: segments are separated by `/`. Escape `~` as `~0` and `/` as `~1`, and percent-encode characters that aren't valid in a URI fragment (e.g. a space becomes `%20`). The pointer to property `"a/b"` is `#/properties/a~1b`.

## 2. Applicators

| Keyword | Meaning / gotchas |
|---|---|
| `allOf` | Must match every subschema. Used for composition and "extends". Pair it with `unevaluatedProperties: false`, not `additionalProperties: false`. |
| `anyOf` | Must match at least one. The default choice for unions. |
| `oneOf` | Must match **exactly one**. Fails if branches overlap (e.g. `{"type":"number"}` and `{"type":"integer"}` both match `1`). Use it only when branches are disjoint, ideally via a `const` discriminator property. |
| `not` | Must not match. Error messages are poor, so use it sparingly. |
| `if` / `then` / `else` | If the instance matches `if`, it must match `then`, otherwise `else`. `if` alone does nothing. Remember that `if: {properties: {kind: {const: "a"}}}` also matches when `kind` is **absent**, so add `required: ["kind"]` inside the `if`. |
| `dependentSchemas` | 2019-09+. `{"credit_card": {"required": ["billing_address"]}}`: if the property is present, apply the schema. |

## 3. Object keywords

| Keyword | Meaning / gotchas |
|---|---|
| `properties` | Schema per named property. Doesn't make them required. |
| `required` | Array of property names that must be present. Belongs **beside** `properties`, not inside a property. Presence only: `null` values count as present. |
| `patternProperties` | Regex → schema, for keys matching the regex (unanchored). |
| `additionalProperties` | Schema for keys not matched by `properties`/`patternProperties` **in the same schema object**. It can't see `allOf`/`$ref` siblings. |
| `propertyNames` | Schema every key must satisfy, e.g. `{"pattern": "^[a-z_]+$"}`. Keys are strings, so don't put `type` here. |
| `minProperties` / `maxProperties` | Count limits. |
| `dependentRequired` | 2019-09+. `{"a": ["b", "c"]}`: if `a` is present, `b` and `c` must be too. |
| `unevaluatedProperties` | See section 6. |

## 4. Array keywords

| Keyword | 2020-12 | 2019-09 / draft-07 |
|---|---|---|
| Tuple positions | `prefixItems: [s0, s1]` | `items: [s0, s1]` (array form) |
| Remaining items | `items: s` (applies after `prefixItems`) | `additionalItems: s` |
| All items (no tuple) | `items: s` | `items: s` |

Others: `minItems`, `maxItems`, `uniqueItems` (deep equality, so `1` and `1.0` are equal), `contains` (at least one item matches), `minContains`/`maxContains` (2019-09+, used together with `contains`, and `minContains: 0` makes `contains` always pass), and `unevaluatedItems` (section 6).

Closed tuple in 2020-12: `{"prefixItems": [{"type":"string"},{"type":"number"}], "items": false}`.

## 5. Validation

Any type: `type` (`"null"`, `"boolean"`, `"object"`, `"array"`, `"number"`, `"string"`, `"integer"`, or an array of these), `enum` (array of allowed values, compared with deep equality), and `const` (a single allowed value).

- `"integer"` accepts any number with a zero fractional part, including `1.0`.
- Nullable is written `"type": ["string", "null"]`, or `anyOf` with `{"type": "null"}` when `$ref` is involved.

Numbers: `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum` (these are **numbers** since draft-06, booleans in draft-04 and OpenAPI 3.0), and `multipleOf` (> 0. Beware floating point: `0.07` fails `multipleOf: 0.01` in both Ajv's default settings and Python `jsonschema`; see their reference files).

Strings: `minLength` and `maxLength` count **Unicode code points**, not bytes or UTF-16 units. `pattern` is ECMA-262 regex and **unanchored**, so write `^...$` for whole-string matches. Avoid lookbehind and named groups if other validators must accept the schema.

## 6. Unevaluated (2019-09+)

`unevaluatedProperties` and `unevaluatedItems` apply to anything not successfully evaluated by **any** adjacent keyword, including through `allOf`, `anyOf`, `oneOf`, `if/then/else`, `$ref`, and `dependentSchemas`. This is the correct way to close a composed schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "allOf": [{ "$ref": "#/$defs/base" }],
  "properties": { "extra": { "type": "string" } },
  "unevaluatedProperties": false,
  "$defs": { "base": { "properties": { "id": { "type": "integer" } } } }
}
```

Subschemas that **fail** don't contribute evaluated properties. In an `anyOf`, only matching branches count.

## 7. Format

`format` names a semantic type: `date-time`, `date`, `time`, `duration`, `email`, `idn-email`, `hostname`, `idn-hostname`, `ipv4`, `ipv6`, `uri`, `uri-reference`, `iri`, `iri-reference`, `uuid`, `uri-template`, `json-pointer`, `relative-json-pointer`, or `regex`.

- In 2019-09 and 2020-12, `format` is an **annotation by default**: validators may ignore it. Ajv checks it only with `ajv-formats`, and Python `jsonschema` only with `format_checker=FormatChecker()`. The bundled scripts assert it unless you pass `--no-format`.
- If the format must be enforced everywhere, add a `pattern` too, or tell the user that other validators may not enforce it.
- Unknown format names are ignored.

## 8. Content

`contentEncoding` (e.g. `"base64"`), `contentMediaType` (e.g. `"application/json"`), and `contentSchema` (2019-09+: the schema for the decoded content). These are annotations only, never asserted by default.

## 9. Meta-data

`title`, `description`, `default` (not applied by validators, it's just information), `examples` (an **array**), `deprecated`, `readOnly`, and `writeOnly`. Code and doc generators use these heavily, so add them to public schemas.

## 10. Recipes

- **Discriminated union**:
  ```json
  {"oneOf": [
    {"properties": {"kind": {"const": "circle"}, "r": {"type": "number"}}, "required": ["kind", "r"]},
    {"properties": {"kind": {"const": "square"}, "side": {"type": "number"}}, "required": ["kind", "side"]}
  ]}
  ```
  For clearer errors, use an `if/then` chain keyed on `kind` instead.
- **Map / dictionary**: `{"type": "object", "additionalProperties": {"$ref": "#/$defs/value"}}`, plus `propertyNames` to constrain keys.
- **Non-empty string**: `{"type": "string", "minLength": 1}`. For trimmed non-empty, add `"pattern": "\\S"`.
- **At least one of A or B**: `{"anyOf": [{"required": ["a"]}, {"required": ["b"]}]}`.
- **Mutually exclusive A and B**: `{"not": {"required": ["a", "b"]}}`.
- **Recursive tree**: `{"$defs": {"node": {"type": "object", "properties": {"children": {"type": "array", "items": {"$ref": "#/$defs/node"}}}}}, "$ref": "#/$defs/node"}`.
