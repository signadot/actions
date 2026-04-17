\description{"Lint Kubernetes manifests with kube-score."}

\image{"zegl/kube-score:v1.9.0"}

Lint \input{manifest, required} against kube-score's built-in checks.
The manifest should be a YAML document (single or multi-document)
containing Kubernetes resource definitions.

\input{options} provides additional kube-score flags as a single
string (e.g. `--ignore-test container-resources`). When omitted,
all default checks run.

\output{exitCode, schema={"type":"integer"}} records kube-score's
exit code: 0 = all checks passed, 1 = one or more checks failed.

\output{report} captures the human-readable score report. Downstream
steps can inspect it for specific failures.

Stdout (the score report) and stderr flow through the runner's log
pipeline.

```sh
set +e
cat ./context/manifest > "$TMPDIR/manifest.yaml"

opts=""
[ -f ./context/options ] && opts="$(cat ./context/options)"

report=$(kube-score score $opts "$TMPDIR/manifest.yaml" 2>&1)
ec=$?

printf '%s' "$report" > ./outputs/report
printf '%d' "$ec" > ./outputs/exitCode
exit "$ec"
```
