\description{"Execute a shell script inside a caller-specified container image."}

\image{ref={"input":"image"}, env={"RUN_IMAGE_DEBUG":"1","FROM_DIRECTIVE":"yes","FROM_INPUT":{"input":"dynamicEnv"}}, timeout={"input":"dynamicTimeout"}}

Execute \input{script, required} inside the container image provided
via \input{image, required}. This is the dynamic-image counterpart
of the `shell` action — the caller picks the image at plan
invocation time.

\input{dynamicEnv, default=""} is exposed as `FROM_INPUT` env var
inside the container; \input{dynamicTimeout, default=""} sets the
wall-clock budget (parseable by Go's time.ParseDuration, e.g. "60s").

All extra_inputs declared on the step are available as named files
in `./context/` — JSON files are parsed as structured values by the
runner's context loader; plain files are read as strings.

\output{exitCode, schema={"type":"integer"}} records the script's
exit code. Non-zero means the script failed.

Stdout and stderr flow through the runner's log pipeline. If a
downstream step needs a structured value, have the script write it
to `./outputs/<name>` and declare it via `extra_outputs`.

**Security:** this action is gated by the org's action policy.

```sh
set +e
echo "[run-image] RUN_IMAGE_DEBUG=$RUN_IMAGE_DEBUG FROM_DIRECTIVE=$FROM_DIRECTIVE FROM_INPUT=$FROM_INPUT"
sh -e ./context/script
ec=$?
printf '%d' "$ec" >./outputs/exitCode
exit "$ec"
```
