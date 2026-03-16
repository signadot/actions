# Signadot Actions

Built-in action definitions for the Signadot platform. The directory layout is
inspired by the [Agent Skills](https://agentskills.io/specification) spec.

Each action is a directory containing an `ACTION.md` file with YAML frontmatter
and a markdown body describing the action's behavior, inputs, outputs, and
executable script blocks.

## ACTION.md Format

### Frontmatter

Standard [Agent Skills](https://agentskills.io/specification) fields (`name`,
`description`, etc.) plus:

| Field      | Description |
| ---------- | ----------- |
| `requires` | Runtime dependencies that must be present on the runner. Each entry is a binary name with an optional semver constraint (e.g. `node >= 20`, `npx`, `playwright`). |

### Body

The body is the action implementation as markdown with fenced code blocks.

**Inline I/O declarations** use `\input{...}` and `\output{...}` syntax:

```
\input{name}
\input{name, required}
\input{name, default="value"}
\input{name, required, schema={"type":"integer","minimum":1}}
```

`\output{...}` supports the same options except `required`.

**Script blocks** are fenced code blocks that define the executable
implementation of the action. Scripts read inputs from `./input/<name>` and
write outputs to `./output/<name>`. **Validation blocks** use the `validation`
language tag and are extracted separately for runner capability checks.

## Actions

| Action | Description |
| ------ | ----------- |
| [`request-http`](request-http/ACTION.md) | Execute an HTTP request and capture the full roundtrip as structured JSON |
| [`check`](check/ACTION.md) | Evaluate an Expr boolean expression against input JSON for pass/fail assertions |
