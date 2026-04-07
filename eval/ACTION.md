\description{"Evaluate an Expr (expr-lang.org) expression against named inputs and return the result."}
\requires{"actionbox"}

Evaluate \input{expression, required} against all named inputs provided to the
step and return the result as \output{result}.

The expression is an [Expr](https://expr-lang.org) expression evaluated against
an environment built from every input file in the context directory. Each file
becomes a top-level variable named by its filename (without extension). JSON
files are parsed as structured values; plain files are read as strings.

This action is a general-purpose expression evaluator. Use it to compose values
from multiple sources (plan params, step outputs, literals) before passing them
to actions that don't evaluate expressions themselves.

**Expression examples:**

| Inputs | Expression | Result |
| ----- | ---------- | ------ |
| `namespace="my-ns"` | `'http://location.' + namespace + ':8081/locations'` | `"http://location.my-ns:8081/locations"` |
| `auth_token="secret"` | `['Authorization: Bearer ' + auth_token]` | `["Authorization: Bearer secret"]` |
| `expected_status="200"` | `'response.statusCode == ' + expected_status` | `"response.statusCode == 200"` |
| `capture={"response":{"statusCode":200}}` | `capture.response.statusCode` | `200` |

**Exit codes:**

- `0` — evaluation completed successfully
- `1` — error (bad flags, malformed JSON, expression compile error)

```sh
actionbox eval \
  --expression "$(cat ./context/expression)" \
  --extra-context ./context \
  > ./outputs/result.json
```
