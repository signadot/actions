# Signadot Actions

Built-in action definitions for the Signadot platform. The directory layout is
inspired by the [Agent Skills](https://agentskills.io/specification) spec.

Each action is a directory containing an `ACTION.md` file with YAML frontmatter
and a markdown body describing the action's behavior, inputs, outputs, and
executable script blocks.

## ACTION.md Format

### Body

The body is the action implementation as markdown with fenced code blocks.
The text in the markdown may contain some directives to specify

- Runner Requirements
- Inputs and Outputs

**Inline I/O declarations** use `\input{...}` and `\output{...}` syntax:

```text
\input{name}
\input{name, required}
\input{name, default="value"}
\input{name, required, schema={"type":"integer","minimum":1}}
\input{name, required, schemaRef="schema/name-schema.json"}
```

`\output{...}` supports the same options except `required`.

`schemaRef` can only refer to files relative to the directory in which
ACTION.md appears. 

**Script blocks** are fenced code blocks that define the executable
implementation of the action. Any code block without a language tag or with `sh` or
`bash` language tags are included in the script blocks of the ACTION.

Scripts read inputs from `./context/<name>` and write outputs to
`./outputs/<name>`. **Validation blocks** use the `validation` language tag and
are extracted separately for runner capability checks.


## Actions

| Action | Description |
| ------ | ----------- |
| [`request-http`](request-http/ACTION.md) | Execute an HTTP request and capture the full roundtrip as structured JSON |
| [`check`](check/ACTION.md) | Evaluate an Expr boolean expression against input JSON for pass/fail assertions |
| [`eval`](eval/ACTION.md) | Evaluate an expression against named inputs and return the result |
| [`shell`](shell/ACTION.md) | Execute a shell script with named inputs from the context directory |
