# Schemas ⇄ code

## Choosing a generator

| Target | Schema → code | Code → schema |
|---|---|---|
| TypeScript types | `json-schema-to-typescript` (CLI `json2ts`) | `ts-json-schema-generator`, `typescript-json-schema` |
| Zod | `json-schema-to-zod` | Zod 4 `z.toJSONSchema()`, or `zod-to-json-schema` for Zod 3 |
| TypeBox | — (TypeBox *is* JSON Schema) | `Type.*` builders emit JSON Schema directly |
| Python Pydantic | `datamodel-code-generator --input-file-type jsonschema` | `Model.model_json_schema()` (Pydantic v2 emits 2020-12) |
| Python dataclasses / TypedDict | `datamodel-code-generator --output-model-type ...` | — |
| Go | `go-jsonschema` (omissis) | `invopop/jsonschema` |
| Java | `jsonschema2pojo` | `victools/jsonschema-generator` |
| C# | NJsonSchema / `quicktype` | NJsonSchema |
| Rust | `typify` | `schemars` |
| Many languages | `quicktype` | — |
| Docs | `json-schema-for-humans`, `@adobe/jsonschema2md` | — |

Check that the generator supports the schema's draft: several (e.g. `json-schema-to-typescript`) target draft-04/07 semantics and may ignore `prefixItems`, `$dynamicRef`, `unevaluated*`, or `dependentRequired`. For a small schema, or when the user can't add a dependency, hand-write the code using the mapping below. After generating, run the project's type checker or tests.

## Type mapping

| Schema | TypeScript | Pydantic v2 | Zod |
|---|---|---|---|
| `{"type":"string"}` | `string` | `str` | `z.string()` |
| `minLength`/`maxLength`/`pattern` | (lost) | `Field(min_length=, max_length=, pattern=)` / `constr` | `.min() .max() .regex()` |
| `format: date-time` | `string` | `datetime` | `z.iso.datetime()` (Zod 4) / `z.string().datetime()` |
| `format: email` / `uuid` / `uri` | `string` | `EmailStr` / `UUID` / `AnyUrl` | `z.email()` / `z.uuid()` / `z.url()` |
| `{"type":"integer"}` | `number` | `int` | `z.number().int()` |
| `{"type":"number"}` + `minimum` | `number` | `float` + `Field(ge=)` | `z.number().min()` |
| `{"type":"boolean"}` | `boolean` | `bool` | `z.boolean()` |
| `{"type":"null"}` / `["T","null"]` | `T \| null` | `T \| None` | `.nullable()` |
| property not in `required` | `prop?: T` | `prop: T \| None = None` (or a default) | `.optional()` |
| `enum: [...]` | union of literals | `Literal[...]` or `Enum` | `z.enum([...])` / `z.union(z.literal...)` |
| `const: x` | literal type | `Literal[x]` | `z.literal(x)` |
| `{"type":"array","items":S}` | `S[]` | `list[S]` | `z.array(S)` |
| `prefixItems: [A,B]`, `items:false` | `[A, B]` | `tuple[A, B]` | `z.tuple([A, B])` |
| `uniqueItems: true` | (lost) | `set[S]` (if hashable) | `.refine(...)` |
| `properties` + `additionalProperties:false` | interface | `model_config = ConfigDict(extra="forbid")` | `z.strictObject({...})` (Zod 4) / `.strict()` |
| `additionalProperties: S` (map) | `Record<string, S>` | `dict[str, S]` | `z.record(z.string(), S)` |
| `allOf: [A, B]` | `A & B` | multiple inheritance / merged model | `A.and(B)` / `.extend()` |
| `anyOf` / `oneOf` | `A \| B` | `A \| B` (`Field(discriminator="kind")` for tagged) | `z.union` / `z.discriminatedUnion("kind", ...)` |
| `$ref: "#/$defs/X"` | named type `X` | class `X` | named const `X` (use `z.lazy` for recursion) |
| `default` | (lost) | `= default` | `.default()` |
| `description` | JSDoc | `Field(description=)` | `.describe()` |
| `readOnly` / `writeOnly` | (lost) | separate input/output models | separate schemas |

## Pydantic strictness trap

Pydantic's default lax mode turns `"4999"` into `4999`, which the schema would reject, so making models strict is tempting. But `model_config = ConfigDict(strict=True)` also rejects ISO date strings for `datetime` fields in `model_validate(dict)`. Only `model_validate_json(text)` parses them in strict mode. Webhook handlers usually call `model_validate(request.json())`, so fully strict models break on real payloads. Prefer per-field strict types (`StrictInt`, `StrictBool`, `StrictStr`) and leave `datetime` fields lax, and test **both** `model_validate(json.loads(text))` and `model_validate_json(text)` against the samples.

## Constructs that don't map cleanly

Flag these to the user instead of silently dropping them:

- `if`/`then`/`else`, `not`, `dependentRequired`/`dependentSchemas`: TS types can't express them. In Pydantic use `@model_validator`, and in Zod `.superRefine`.
- `patternProperties`, `propertyNames`: TS falls back to an index signature, and Pydantic to `dict` plus a validator.
- `unevaluatedProperties`: usually approximate it as "closed object" over the merged properties.
- `$dynamicRef`: generators rarely support it. Resolve it to a concrete type by hand.
- `multipleOf`, `minContains`, `contentSchema`: runtime validation only.

A common robust pattern is to generate *types* for compile-time help and keep validating at runtime against the real schema with Ajv or `jsonschema`. That way the schema stays the single source of truth.
