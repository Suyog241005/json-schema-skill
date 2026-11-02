# Python `jsonschema` (4.x) + `referencing`

Install with `pip install jsonschema`. For full format checking, use `pip install "jsonschema[format]"`, which adds the `rfc3339-validator`, `fqdn`, `idna`, `rfc3987` and similar dependencies some formats need. It supports draft-03 through 2020-12 and follows the spec closely, including ignoring `$ref` siblings in draft-07 and earlier.

## Core API

```python
from jsonschema import Draft202012Validator, validators, FormatChecker
from jsonschema.exceptions import ValidationError, SchemaError, best_match

Draft202012Validator.check_schema(schema)              # raises SchemaError if the schema itself is invalid
cls = validators.validator_for(schema)                 # class from $schema (default: latest)
v = cls(schema, format_checker=FormatChecker())        # format is ignored unless you pass a checker
errors = sorted(v.iter_errors(data), key=lambda e: list(e.absolute_path))
for e in errors:
    print("/".join(map(str, e.absolute_path)) or "(root)", e.message, list(e.absolute_schema_path))
v.is_valid(data)                                       # bool
raise best_match(v.iter_errors(data))                  # the single most relevant error, for users
```

- `jsonschema.validate(data, schema)` is the one-shot form. It raises `ValidationError` on the first (best) error and re-checks the schema every call, so build a validator object for repeated use.
- On a `ValidationError`, `.message`, `.absolute_path` (instance location, a deque), `.absolute_schema_path`, `.validator` (keyword), `.validator_value`, and `.instance` are useful. `.context` holds the sub-errors of `anyOf`/`oneOf` branches.
- `validators.extend(Draft202012Validator, {"even": fn})` adds a custom keyword, where `fn(validator, value, instance, schema)` yields `ValidationError`s.
- `FormatChecker()` with `@checker.checks("semver")` registers a custom format.

## Multi-file schemas (`referencing`)

`RefResolver` is deprecated. Use a `referencing.Registry`:

```python
import json
from pathlib import Path
from referencing import Registry, Resource
from referencing.jsonschema import DRAFT202012

def retrieve(uri: str):
    path = Path("schemas") / uri.rsplit("/", 1)[-1]          # map URIs to local files however fits the project
    return Resource.from_contents(json.loads(path.read_text()), default_specification=DRAFT202012)

registry = Registry(retrieve=retrieve)
# or pre-load: Registry().with_resources([(s["$id"], Resource.from_contents(s)) for s in schemas])
v = Draft202012Validator(root_schema, registry=registry)
```

`Resource.from_contents` reads `$schema` to pick the spec, and `default_specification` applies when it's missing. The library never fetches remote refs by itself. If needed, do the fetching inside `retrieve`, and cache it.

## Gotchas

- **`format` is not checked** unless you pass `format_checker=`, and some formats silently pass if their optional dependency isn't installed. Use the `[format]` extra.
- **Float `multipleOf`**: stock `jsonschema` (like Ajv's default) says `0.3` is not a multiple of `0.1`, because of float division. In a user's own code, override the keyword with an exact decimal check. Use this whenever decimal `multipleOf` values are involved:
  ```python
  from decimal import Decimal
  from jsonschema import validators
  from jsonschema.exceptions import ValidationError
  Base = validators.validator_for(schema)
  stock = Base.VALIDATORS["multipleOf"]
  def multiple_of(v, m, instance, s):
      if isinstance(instance, float) or isinstance(m, float):
          if v.is_type(instance, "number") and Decimal(str(instance)) % Decimal(str(m)) != 0:
              yield ValidationError(f"{instance!r} is not a multiple of {m}")
          return
      yield from stock(v, m, instance, s)
  Validator = validators.extend(Base, {"multipleOf": multiple_of})
  ```
  Avoid `json.loads(parse_float=Decimal)` as the fix: it makes `1.0` stop counting as an `integer`.
- Python `True`/`False` are not numbers to the validator, and `1.0` is an integer. That matches the spec.
- Pydantic's `model_json_schema()` emits 2020-12 but with `$defs` refs and its own `title`s. Validate data with the validator, not by round-tripping through models.
