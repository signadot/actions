\description{"Execute a shell script inside a caller-specified container image."}

\image{ref={"input":"image"}}
\timeout{{"input":"timeout"}}

Execute \input{script, required} inside the container image provided
via \input{image, required}. The caller picks the image at plan
invocation time; the image is required.

\input{timeout, default=""} bounds the script's wall clock. The value
is parsed by Go's `time.ParseDuration` (e.g. `"60s"`, `"5m"`). Empty
falls through to the runner's system default cap.

All extra_inputs declared on the step are available as named files
in `./context/` — JSON files are parsed as structured values by the
runner's context loader; plain files are read as strings. Bridge them
to env vars in the script if needed:

```text
[ -f ./context/base_url ] && export BASE_URL=$(cat ./context/base_url)
```

\output{exit_code, schema={"type":"integer"}} records the script's
exit code. Non-zero means the script failed.

Stdout and stderr flow through the runner's log pipeline. If a
downstream step needs a structured value, have the script write it
to `./outputs/<name>` and declare it via `extra_outputs`.

**Security:** this action is gated by the org's action policy.

## Authoring rules

The schema on each `extra_input` picks the file the script reads:

- **with schema** → `./context/<name>.json` (raw JSON bytes; consume
  with `jq`, `python -c "import json"`, etc.)
- **without schema** → `./context/<name>` (raw text; JSON string values
  are unquoted, so `cat ./context/<name>` gives the string directly)

Anything the script reads from `./context/` MUST be declared as an
`extra_input` and wired via `refs` — files don't appear automatically.

**Routing env vars come from `routingContext`, not from plan params.**
When the step has `routingContext` set, the runner injects whichever of
`SIGNADOT_ROUTING_KEY`, `SIGNADOT_SANDBOX_NAME`, and
`SIGNADOT_ROUTEGROUP_NAME` apply to the resolved target. Read them
directly with `$VAR` — do **not** declare a plan param like `routing_key`
and pass it through `extra_inputs` so the script reads it from
`./context/`. (A plan param named `sandbox` that *drives*
`routingContext` via `{ref: {sandboxRef: params.sandbox}}` is fine; what
isn't fine is a plan param the script reads as a routing key.)

```sh
set +e
sh -e ./context/script
ec=$?
printf '%d' "$ec" >./outputs/exit_code
exit "$ec"
```
