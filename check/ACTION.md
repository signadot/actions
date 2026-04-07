\description{"Evaluate a boolean expression against named inputs and produce a pass/fail result."}
\requires{"actionbox"}

Evaluate \input{expression, required} against all named inputs provided to the
step and produce \output{result} indicating pass or fail. \input{name, required}
identifies the check in results.

The expression is an [Expr](https://expr-lang.org) boolean expression evaluated
against an environment built from every input file in the context directory.
Each file becomes a top-level variable named by its filename (without extension).
JSON files are parsed as structured values; plain files are read as strings.
Non-boolean expressions are rejected at compile time.

\input{results_file} is a path to append results in NDJSON format. Multiple
invocations can append to the same file.

**Expression examples:**

| Inputs | Expression |
| ----- | ---------- |
| `capture={"response":{"statusCode":200}}` | `capture.response.statusCode == 200` |
| `capture={"response":{"message":{"body":"invalid"}}}` | `capture.response.message.body contains "invalid"` |
| `capture=..., expected_status=200` | `capture.response.statusCode == expected_status` |
| `capture=..., auth_token="secret"` | `auth_token in capture.response.message.body` |

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
    "message": "expression \"capture.response.statusCode == 200\" evaluated to false"
  }
}
```

```sh
actionbox check --context-dir ./context > ./outputs/result.json
```
