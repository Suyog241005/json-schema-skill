# Ajv (v8) for application code

Install with `npm install ajv ajv-formats`, plus `ajv-draft-04` for draft-04 and `ajv-errors` for custom messages.

## Contents
- Pick the class for the draft
- Compile and validate
- Options worth knowing
- Errors
- Multi-file schemas and remote refs
- Formats
- Custom keywords and formats
- Standalone (precompiled) validators
- Known gaps and deviations

## Pick the class for the draft

| Draft | Import |
|---|---|
| 2020-12 | `import Ajv2020 from "ajv/dist/2020.js"` (CJS: `require("ajv/dist/2020").default`) |
| 2019-09 | `import Ajv2019 from "ajv/dist/2019.js"` |
| draft-07 (default), draft-06 | `import Ajv from "ajv"`, and for draft-06 also `ajv.addMetaSchema(require("ajv/dist/refs/json-schema-draft-06.json"))` |
| draft-04 | `import Ajv from "ajv-draft-04"` |

A single instance can't mix 2020-12 with draft-07 or 2019-09 schemas. Use one instance per draft family. The most common Ajv error, `no schema with key or ref "https://json-schema.org/draft/2020-12/schema"`, means a 2020-12 schema went to the default `Ajv` (draft-07) class.

## Compile and validate

```js
import Ajv2020 from "ajv/dist/2020.js";
import addFormats from "ajv-formats";

const ajv = new Ajv2020({ allErrors: true });
addFormats(ajv);
const validate = ajv.compile(schema);     // compile once, reuse; compiling is the expensive part
if (!validate(data)) console.log(validate.errors);
```

`ajv.validateSchema(schema)` meta-validates without compiling. `ajv.addSchema(schema, key?)` registers a schema for `$ref`s, and `ajv.getSchema(id)` gets a compiled validator by `$id`.

## Options worth knowing

- `allErrors: true` reports all errors, not just the first. Turn it on for user-facing validation, but leave it off for untrusted input at scale, because it costs more.
- `strict` defaults to `true`, so unknown keywords, ignored `additionalItems`, and similar problems **throw at compile time**. Individual parts can be set to `"log"` or `false`: `strictSchema` (unknown keywords), `strictTypes`, `strictTuples`, `strictRequired`. Third-party schemas often fail strict mode, so use `strict: false` for them, not edits to the schema.
- `useDefaults: true` fills in `default` values (this mutates the data). `coerceTypes: true` converts `"1"` → `1` (also mutating). `removeAdditional: "all" | true | "failing"` strips extra properties. These are not spec behavior, so call it out when a project relies on them.
- `multipleOfPrecision: N` fixes float `multipleOf` (by default `0.07` fails `multipleOf: 0.01`).
- `discriminator: true` enables the OpenAPI `discriminator` keyword, which gives faster `oneOf` with better errors.
- `$data: true` enables `{"$data": "1/otherField"}` references (non-standard).

## Errors

Each error is `{ instancePath, schemaPath, keyword, params, message }`, e.g. `{ instancePath: "/items/0/qty", keyword: "minimum", params: { comparison: ">=", limit: 1 }, message: "must be >= 1" }`.

- `instancePath` is a JSON Pointer, with `""` for the root. For `required` errors, the missing name is in `params.missingProperty`, and for `additionalProperties` in `params.additionalProperty`.
- `ajv.errorsText(validate.errors, { separator: "\n" })` gives a quick string.
- `anyOf`/`oneOf` failures list every branch's errors plus the combinator error. To explain them, group by `instancePath` and start with the deepest. For good `oneOf` errors, use a `const` discriminator with `if/then`, or `discriminator: true`.
- `ajv-errors` (`import ajvErrors from "ajv-errors"`, needs `allErrors: true`) adds an `errorMessage` keyword for custom messages per property or keyword.

## Multi-file schemas and remote refs

- Give each schema a `$id` and call `ajv.addSchema(otherSchema)` before compiling the root. `$ref: "address.json"` resolves against the root's `$id`.
- Without `$id`s, register under a key with `ajv.addSchema(schema, "address.json")`. Relative resolution then depends on the root's base, so prefer real `$id`s.
- Remote refs: `new Ajv2020({ loadSchema: async (uri) => (await fetch(uri)).json() })`, then `await ajv.compileAsync(schema)`. Plain `compile` throws `can't resolve reference`.
- Ajv has no bundler. Embed referenced schemas under `$defs` by hand, keyed by their `$id` (see SKILL.md), or use a library like `@apidevtools/json-schema-ref-parser`. Note that ref-parser's `bundle()` rewrites `$ref`s to internal pointers rather than embedding `$id` resources.

## Formats

`ajv-formats` adds `date`, `time`, `date-time`, `duration`, `uri`, `uri-reference`, `uri-template`, `email`, `hostname`, `ipv4`, `ipv6`, `regex`, `uuid`, `json-pointer`, `relative-json-pointer`, `byte`, `int32`, `int64`, `float`, `double`, `password`, and `binary`. `addFormats(ajv, ["date", "email"])` adds only some, and `addFormats(ajv, { mode: "fast" })` uses simpler regexes. Without `ajv-formats`, strict mode throws on `unknown format "email"`, and non-strict mode ignores it.

## Custom keywords and formats

```js
ajv.addFormat("semver", /^\d+\.\d+\.\d+$/);
ajv.addKeyword({
  keyword: "even",
  type: "number",
  schemaType: "boolean",
  validate: (schemaValue, data) => !schemaValue || data % 2 === 0,
  errors: false
});
ajv.addKeyword("x-internal"); // declare an annotation-only keyword so strict mode accepts it
```

## Standalone (precompiled) validators

`import standaloneCode from "ajv/dist/standalone/index.js"`: compile with `new Ajv2020({ code: { source: true, esm: true } })` and write `standaloneCode(ajv, validate)` to a `.js` file. This gives validators without runtime compiling, which suits Cloudflare Workers and other CSP-restricted environments where `new Function` is blocked.

## Known gaps and deviations

- **`$ref` siblings in draft-07/06/04 are applied**, while the spec says to ignore them. This hides migration differences, so see `drafts.md`.
- `$dynamicRef` is supported for the common recursive-extension patterns, but complex dynamic-scope cases can differ from the spec. `$recursiveRef` (2019-09) works.
- Unicode string lengths count code points (default `unicode: true`), which matches the spec.
- The `format` assertion vocabulary and `$vocabulary` are only partially honored.
- Ajv doesn't parse YAML. Parse it with `yaml` and pass the object.
