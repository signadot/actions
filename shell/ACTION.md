\description{"Execute a shell script with named inputs from the context directory."}

Execute \input{script, required} as a shell script. All extra_inputs declared
on the step are available as named files in `./context/` — JSON files are
parsed as structured values by the runner's context loader; plain files are
read as strings.

\output{exitCode, schema={"type":"integer"}} records the script's numeric exit
status. It is exposed as a named output — rather than relying solely on the
runner's step-level success/failure — so downstream steps can branch on
specific non-zero values without causing the shell step itself to fail.

The script's stdout and stderr flow through the process's own streams and are
captured by the runner's log pipeline; view them with `signadot plan x logs`
(list, download, or follow with `-f`). They are intentionally NOT named outputs:
if a downstream step needs a structured value derived from the script, have the
script write it to `./outputs/<name>` and declare it via `extra_outputs`. The
runner reads extra outputs back according to their schema declarations.

This action is a general-purpose shell executor. Use it when the task requires
command-line tools (`curl`, `jq`, `grep`, etc.) or logic that Expr expressions
cannot express. Inputs from plan params or step outputs arrive via `extra_inputs`
wired through `refs`.

**Security:** this action is gated by the org's action policy. It is not
available unless the org admin has explicitly enabled it. The compile path
filters it from the available actions list when it is not enabled.

**Script source:** when the user's prompt contains a fenced ` ```sh ` or
` ```bash ` code block, the compile path extracts the code verbatim and
injects it as `values.script`. The LLM generates the step wiring (id,
extra_inputs, refs) but does not modify the user's code.

**Exit codes:**

- `0` — script completed successfully
- non-zero — script failed (exit code propagated)

```sh
set +e
sh -e ./context/script
ec=$?
printf '%d' "$ec" >./outputs/exitCode
exit "$ec"
```
