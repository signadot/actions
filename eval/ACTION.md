\description{"Evaluate an Expr expression against named inputs and return the result."}
\requires{"actionbox"}
\extra_inputs_schema{default={}}

Evaluate \input{expression, required} against all named inputs provided to the
step and return the result as \output{result}.

The expression is an [Expr](https://expr-lang.org) expression evaluated against
an environment built from every input file in the context directory. Each file
becomes a top-level variable named by its filename (without extension). JSON
files are parsed as structured values; plain files are read as strings.

**Extra_input schemas.** Extra_inputs on this action default to `{}` (the
permissive "accept any" JSON Schema) when the plan author doesn't declare
one — the compile pass fills them in. That keeps the input routed to
`./context/<name>.json` so the expression evaluator sees a properly-typed
value. Declaring a specific schema on an extra_input is still useful when
the shape is known: it preserves richer type info for documentation and
future type-aware tooling.

**Expression examples:**

Each row assumes the input is declared as an `extra_input` on the step. The
schema is optional — omit it to let the action's `default={}` fill in, or
declare one like `{"type": "string"}` when the shape is known.

| Inputs | Expression | Result |
| ----- | ---------- | ------ |
| `namespace="my-ns"` | `'http://location.' + namespace + ':8081/locations'` | `"http://location.my-ns:8081/locations"` |
| `auth_token="secret"` | `['Authorization: Bearer ' + auth_token]` | `["Authorization: Bearer secret"]` |
| `expected_status="200"` | `'response.statusCode == ' + expected_status` | `"response.statusCode == 200"` |
| `capture={"response":{"statusCode":200}}` | `capture.response.statusCode` | `200` |

**Exit codes:**

- `0` — evaluation completed successfully
- `1` — error (bad flags, malformed JSON, expression compile error)

## Authoring rules

This action is a general-purpose expression evaluator. Use it to compose values
from multiple sources (plan params, step outputs, literals) before passing them
to actions that don't evaluate expressions themselves.

**When NOT to use `eval`:**

- **For static values.** If the value is a constant known at authoring time
  (a fixed URL, a literal number or boolean, a hardcoded JSON object), pass
  it directly via `args.values` on the consuming step. An `eval` step that
  emits a constant is dead weight.
- **To parameterize a `check`.** `check` evaluates expressions natively
  against its `extra_inputs`. Bring plan params and step outputs into the
  check expression by declaring them as `extra_inputs` on the *check* step
  and referencing them in the expression — do not insert an `eval` step in
  front of the check.
- **To pull a sub-field from a structured value** that already declares a
  schema. Use a drill-in ref (`steps.X.outputs.Y.field` or
  `params.X.field`) directly on the consumer step.

Reach for `eval` only when a composed value must reach an action that
does NOT evaluate expressions itself (e.g. building a URL string or a
headers array for `request-http`).

```sh
actionbox eval \
  --expression "$(cat ./context/expression)" \
  --extra-context ./context \
  > ./outputs/result.json
```
