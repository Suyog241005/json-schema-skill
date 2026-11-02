---
name: json-schema
description: Write, validate, review, fix, migrate, bundle, and generate code from JSON Schema (json-schema.org), defaulting to draft 2020-12 and supporting 2019-09 (plus draft-07/06/04 when a file declares them). Validates with Ajv (Node) or Python jsonschema. Use this skill whenever the user mentions JSON Schema, a `$schema`/`$ref`/`$defs` keyword, a `*.schema.json` file, Ajv or the Python jsonschema library, validating JSON/config/API payloads against a schema, inferring a schema from sample JSON, converting schemas to or from TypeScript/Pydantic/Zod types, OpenAPI component schemas, or asks "why does this JSON fail validation", even if they never say "JSON Schema" explicitly.
---

# JSON Schema

## Defaults

- **Dialect**: write new schemas as **2020-12** (`"$schema": "https://json-schema.org/draft/2020-12/schema"` as the first key). Use **2019-09** when the user or project uses it. Keep an existing file's draft unless asked to migrate.
- **Validator**: use **Ajv** (Node) or **Python `jsonschema`**. When the project already depends on one (check `package.json`, `requirements.txt`, `pyproject.toml`), use that one, because they care how *their* validator behaves. Otherwise use Ajv as shown below.
- **Verify, don't eyeball.** After writing or changing a schema, run a validator against sample data, including a few invalid instances you make up. The point is to prove it rejects what it should, not just that it accepts the samples.
- **Reply concisely.** Lead with the answer: what's broken or changed, and anything the user must decide. Don't restate unchanged parts, paste validator logs, or list every check you ran. One line saying what you verified is enough.

## Validating

Install the validator in a throwaway directory, not in the user's project: `mkdir -p /tmp/jsv && cd /tmp/jsv && npm i -s ajv@8 ajv-formats@3`. Then save this as `/tmp/jsv/check.mjs` and run `node /tmp/jsv/check.mjs schema.json data1.json data2.json [--ref other.schema.json]`:

```js
// usage: node check.mjs schema.json [instance.json ...] [--ref other.schema.json ...]
import { readFileSync } from "node:fs";
import Ajv2020 from "ajv/dist/2020.js";          // 2019-09: ajv/dist/2019.js · draft-07: ajv
import addFormats from "ajv-formats";
const args = process.argv.slice(2), refs = [], files = [];
for (let i = 0; i < args.length; i++) args[i] === "--ref" ? refs.push(args[++i]) : files.push(args[i]);
const [schemaFile, ...instances] = files;
const load = (f) => JSON.parse(readFileSync(f, "utf8"));
const ajv = new Ajv2020({ allErrors: true, strict: false, strictSchema: true, multipleOfPrecision: 12 }); // unknown keywords throw
addFormats(ajv);
for (const r of refs) ajv.addSchema(load(r), r);   // key = path, so "$ref": "address.json" resolves
const validate = ajv.compile(load(schemaFile));     // throws on invalid schema or unknown keyword
let bad = 0;
for (const f of instances) {
  if (validate(load(f))) console.log("VALID  ", f);
  else { bad++; console.log("INVALID", f); for (const e of validate.errors) console.log(`  ${e.instancePath || "(root)"}: ${e.message}`); }
}
process.exit(bad ? 1 : 0);
```

- `strictSchema: true` makes compiling **throw on unknown or misspelled keywords** (`require`, `minLenght`, `tpye`), which validators otherwise silently ignore. Ajv reports one at a time, so fix and re-run until it compiles. Keep the other strict options off: they reject valid schemas (e.g. `properties` without `type`).
- `multipleOfPrecision` fixes Ajv's float bug (`0.3` vs `multipleOf: 0.1`). `ajv-formats` makes `format` (e.g. `email`, `date-time`) actually checked.
- For 2019-09 import `ajv/dist/2019.js`; for draft-07/06 import `ajv`; for draft-04 install and import `ajv-draft-04`. One Ajv instance can't mix 2020-12 with older drafts.

**Python `jsonschema`** follows the spec where Ajv doesn't (see the table below), so use it in Python projects and to get spec-accurate draft-07 results. Put it in a throwaway venv: `python3 -m venv /tmp/jsonschema-venv && /tmp/jsonschema-venv/bin/pip install -q jsonschema`. Then use `validators.validator_for(schema)(schema, format_checker=FormatChecker()).iter_errors(data)`. `references/python-jsonschema.md` has the multi-file `Registry` setup and the exact-`multipleOf` override.

When explaining errors, translate them into plain language: which field, what's wrong, and what's expected. A pile of errors under one `oneOf`/`anyOf` usually has a single cause (often one bad value in the branch the data was meant to match), so name that cause.

## Validator differences that change results

This is the part people get wrong. Check it whenever results look surprising, differ between tools, or a migration is involved:

| Behavior | Spec | Ajv 8 | Python `jsonschema` |
|---|---|---|---|
| Keywords next to `$ref` in draft-07 and earlier | ignored | **applied** | ignored |
| `format` | annotation only (2019-09+) | ignored unless `ajv-formats` is added | ignored unless `format_checker=` is passed |
| Unknown keywords | ignored | **compile error** by default (`strict: true`) | ignored |
| `multipleOf: 0.1` with `0.3` | valid | **invalid** unless `multipleOfPrecision` is set | **invalid** (float division) unless you override `multipleOf` |
| Mixed drafts across `$ref`s | allowed | not in one instance | allowed |
| Remote `$ref`s | implementation-defined | only with `loadSchema` + `compileAsync` | only via a `referencing` `Registry` retrieve function |

If the user's own code validates with stock `jsonschema` or Ajv, the float `multipleOf` problem bites them too, so point that out and show the fix (see the reference files).

## Easy-to-miss schema bugs

Compile with `strictSchema: true` first: it catches misspelled keywords like `require`, `minLenght`, and `tpye`, which validators otherwise silently ignore. Beyond typos, these are the bugs worth checking:
- `additionalProperties: false` next to `allOf`/`$ref` can't see the properties those pull in, so it rejects valid data. Use `unevaluatedProperties: false` instead (2019-09+).
- `oneOf` with overlapping branches (e.g. a discriminator property that isn't `required` in each branch). For readable errors on tagged unions, use `if/then` keyed on the required tag. Each `if` should also require the tag, so data missing it doesn't fall through every branch.
- Unanchored `pattern` (it matches anywhere in the string), boolean `exclusiveMinimum` (draft-04/OpenAPI 3.0 style), and `#/definitions/` refs in a schema whose definitions moved to `$defs`.
- `if` without `required` inside it, because `{"properties": {"kind": {"const": "a"}}}` also matches when `kind` is missing.
- The 2020-12 meta-schema still accepts legacy `definitions`/`dependencies`, so strict mode won't flag them.

## Task notes

- **Migrating drafts**: use the mapping in `references/drafts.md`. Validate the examples against the old *and* new schema and report every changed result. The classic change is `$ref` siblings, which draft-07 ignores but 2019-09+ applies. Ajv applied them all along, so record the "before" results with Python `jsonschema`, and say which validator the user's outcome depends on. If a sibling was dead code, ask or state whether you kept it (it now applies) or dropped it (behavior stays the same).
- **Bundling**: copy each referenced schema into the root's `$defs`, keyed by its `$id`, and keep that `$id` inside it. Give a schema without one an `$id` equal to the path its `$ref`s use, and leave the `$ref`s unchanged (this is the 2020-12 bundling approach). Then re-validate the examples against the bundle copied alone into an empty directory, to prove it's self-contained. Replace placeholder `$id`s before publishing.
- **OpenAPI 3.0 → 2020-12**: convert `nullable` to type arrays (and add `null` to any `enum`), boolean `exclusiveMinimum` to numeric, `example` to `examples`, and drop `discriminator` (keep a `const`/`enum` tag). The full table is in `references/drafts.md`.
- **Code generation**: see `references/codegen.md` (read its Pydantic strictness note before setting `strict=True`). Flag constructs that don't map cleanly to types (`if/then`, `not`, `patternProperties`, `unevaluated*`), and keep runtime validation against the real schema.

## Reference files (read only the one the task needs)

- `references/keywords.md`: all 2020-12/2019-09 keywords with gotchas and recipes.
- `references/drafts.md`: draft-04 → 07 → 2019-09 → 2020-12 mapping, and OpenAPI 3.0 conversion.
- `references/ajv.md`: Ajv in app code (classes per draft, options, errors, multi-file refs, formats, custom keywords, standalone code).
- `references/python-jsonschema.md`: `jsonschema` in app code (validators, errors, `referencing` registries, formats, the `multipleOf` fix).
- `references/codegen.md`: schema ⇄ TypeScript/Zod/Pydantic/Go mappings and generators.
