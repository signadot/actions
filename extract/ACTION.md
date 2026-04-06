\description{"Extract a value from a JSON object using an expression."}
\requires{"actionbox"}

Evaluate \input{expression, required} against \input{object, required} and
return the extracted value as \output{result}.

The expression is an [Expr](https://expr-lang.org) expression evaluated against
the input JSON object. The object is parsed as JSON and its top-level keys
become variables in the expression. The result can be any JSON-serializable
value (string, number, object, array, boolean, or null).

**Expression examples:**

| Input | Expression | Result |
| ----- | ---------- | ------ |
| `{"response": {"statusCode": 200}}` | `response.statusCode` | `200` |
| `{"items": [{"name": "a"}, {"name": "b"}]}` | `items[0].name` | `"a"` |
| `{"body": {"id": 1, "name": "foo"}}` | `body` | `{"id": 1, "name": "foo"}` |
| `{"values": [1, 2, 3]}` | `len(values)` | `3` |

**Exit codes:**

- `0` — extraction completed successfully
- `1` — error (bad flags, malformed JSON, expression compile error)

```sh
set -- \
  --expression "$(cat ./context/expression)" \
  --input ./context/object

actionbox extract "$@" > ./outputs/result.json
```
