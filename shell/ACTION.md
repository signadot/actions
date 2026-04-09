\description{"Execute a shell script with named inputs from the context directory."}

Execute \input{script, required} as a shell script. All extra_inputs declared
on the step are available as named files in `./context/` — JSON files are
parsed as structured values by the runner's context loader; plain files are
read as strings.

The action captures the process's natural outputs:
\output{stdout} and \output{stderr} as text,
and \output{exitCode, schema={"type":"integer"}} as an integer.

Any additional structured data the script writes to `./outputs/<name>` is
exposed via `extra_outputs` declared on the step — the runner reads them
back according to their schema declarations.

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
sh -e ./context/script >./outputs/stdout 2>./outputs/stderr
ec=$?
printf '%d' "$ec" >./outputs/exitCode
exit "$ec"
```
