\description{"Run a container image with named inputs and capture outputs."}

Run \input{image, required} as a container with the step's named inputs
mounted into the container's context directory. All extra_inputs declared on
the step are available as named files in `/context/` inside the container —
JSON files at `/context/<name>.json`, plain files at `/context/<name>`.

\input{command} overrides the image's default entrypoint.
\input{args} provides arguments to the command as a JSON array of strings.
\input{env} provides environment variables as a JSON object of `key: value`
string pairs.

The container's `/outputs/` directory is mapped back to the step's outputs.
Write results to `/outputs/<name>` (raw) or `/outputs/<name>.json` (JSON) and
declare them via `extra_outputs` on the step. The runner reads extra outputs
back according to their schema declarations.

\output{exitCode, schema={"type":"integer"}} records the container's exit
status. It is exposed as a named output so downstream steps can branch on
specific non-zero values without causing the step itself to fail.

The container's stdout and stderr flow through the runner's log pipeline;
view them with `signadot plan x logs`. They are intentionally NOT named
outputs: if a downstream step needs a structured value, have the container
write it to `/outputs/<name>.json` and declare it via `extra_outputs`.

**Security:** this action is gated by the org's action policy. It is not
available unless the org admin has explicitly enabled it.

**Exit codes:**

- `0` — container completed successfully
- non-zero — container failed (exit code propagated)

```sh
set -- --image "$(cat ./context/image)"

[ -f ./context/command ] && set -- "$@" --command "$(cat ./context/command)"
[ -f ./context/args.json ] && set -- "$@" --args-file ./context/args.json
[ -f ./context/env.json ] && set -- "$@" --env-file ./context/env.json

actionbox run-image "$@" \
  --context-dir ./context \
  --output-dir ./outputs

ec=$?
printf '%d' "$ec" >./outputs/exitCode
exit "$ec"
```
