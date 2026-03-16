---
name: check
description: >
  Evaluate an Expr boolean expression against input JSON and produce a
  pass/fail result. Use when asserting conditions on HTTP responses,
  test outputs, or any structured data.
requires:
  - actionbox
---

Evaluate \input{expression, required} against \input{input, required} and
produce \output{result} indicating pass or fail. \input{name, required}
identifies the check in results.

The expression is an [Expr](https://expr-lang.org) boolean expression evaluated
against the input JSON. The input is parsed as a JSON object and its top-level
keys become variables in the expression. Non-boolean expressions are rejected at
compile time.

\input{attrs} is a list of `key=value` attributes to include in error output
for additional context.

\input{results_file} is a path to append results in NDJSON format. Multiple
invocations can append to the same file.

**Expression examples:**

| Input | Expression |
| ----- | ---------- |
| `{"status_code": 200}` | `status_code == 200` |
| `{"body": "invalid quantity"}` | `body contains "invalid"` |
| `{"items": [{"name": "a"}, {"name": "b"}]}` | `len(items) > 0 and all(items, {.name != ""})` |
| `{"error_rate": 3.2, "threshold": 5}` | `error_rate <= threshold` |
| `{"body": {"id": 1, "name": "foo"}}` | `"id" in keys(body) and "name" in keys(body)` |

**Exit codes:**

- `0` — check completed (passed or failed — inspect the result JSON)
- `1` — error (bad flags, malformed JSON, expression compile error)

**Output format:**

\output{result, schema=schemas/result.json} is a `TestError` JSON object.

Pass:

```json
{"check": "status is 200"}
```

Fail:

```json
{
  "check": "status is 200",
  "error": {
    "message": "expression \"response.statusCode == 200\" evaluated to false",
    "attrs": {"endpoint": "/api/items"}
  }
}
```

```sh
args=(
  --name "$(cat ./input/name)"
  --expression "$(cat ./input/expression)"
  --input ./input/input
)

[ -f ./input/results_file ] && args+=(--results-file "$(cat ./input/results_file)")

if [ -f ./input/attrs ]; then
  while IFS= read -r attr; do
    args+=(--attr "$attr")
  done < ./input/attrs
fi

actionbox check "${args[@]}" > ./output/result
```
