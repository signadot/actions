\description{"Evaluate an Expr boolean expression against named inputs and produce a pass/fail result."}
\requires{"actionbox"}
\extra_inputs_schema{default={}}

Evaluate \input{expression, required} against the step's named inputs and
produce a `result` indicating pass or fail. \input{name, required}
identifies the check in results.

The expression is an [Expr](https://expr-lang.org) boolean expression evaluated
against an environment built from every input file in the context directory.
Each file becomes a top-level variable named by its filename (without
extension): JSON files are parsed as structured values; plain files are read as
strings. The reserved names `name`, `expression`, `results_file`, and `attrs`
are consumed by check itself and do NOT appear in the expression env.

**Extra_input schemas.** Extra_inputs on this action default to `{}` (the
permissive "accept any" JSON Schema) when the plan author doesn't declare
one — the compile pass fills them in. That keeps the input routed to
`./context/<name>.json` so the expression evaluator sees a properly-typed
value. Declaring a specific schema on an extra_input is still useful when
the shape is known: it preserves richer type info for documentation and
future type-aware tooling.

Non-boolean expressions are rejected at compile time.

\input{attrs, schemaRef="schemas/attrs.json"} is a JSON object of `key: value`
attributes that, when the check fails, are attached to the failure record for
additional context. Pass it as a step value (or via a ref to a JSON object).

\input{results_file} is a path to append results in NDJSON format. Multiple
invocations can append to the same file.

**Expression examples:**

Assuming the step declares `extra_inputs: [{"name": "capture", "schema": {}}]`
and refs `capture` to a previous step's HTTP capture:

| Expression | Description |
| ---------- | ----------- |
| `capture.response.statusCode == 200` | Status code check |
| `capture.response.statusCode in [200, 201, 204]` | Multiple acceptable codes |
| `len(capture.response.message.body.items) > 0` | Non-empty body items |
| `capture.response.message.body.error == nil` | No error in body |

**Exit codes:**

- `0` — check completed (passed or failed — inspect the result JSON)
- `1` — error (bad flags, malformed JSON, expression compile error)

**Output format:**

\output{result, schemaRef="schemas/result.json"} is a `TestError` JSON object.

Pass:

```json
{"check": "status is 200"}
```

Fail:

```json
{
  "check": "status is 200",
  "error": {
    "message": "expression \"capture.response.statusCode == 200\" evaluated to false",
    "attrs": {"endpoint": "/api/items"}
  }
}
```

## Authoring rules

Bring plan params and step outputs into the expression by declaring them as
`extra_inputs` on the step and wiring them via `refs`. Reference the inputs
by their declared names from inside the expression.

**Do not insert an `eval` step in front of a check.** The `check` action
evaluates expressions natively, so plan params and step outputs that the
expression needs should be declared as `extra_inputs` on the check step
itself and referenced by name in the expression. An `eval` step that
just produces an expression string for `check` to evaluate is redundant.

```sh
set -- \
  --name "$(cat ./context/name)" \
  --expression "$(cat ./context/expression)" \
  --context-dir ./context

[ -f ./context/results_file ] && set -- "$@" --results-file "$(cat ./context/results_file)"

actionbox check "$@" > ./outputs/result.json
```
