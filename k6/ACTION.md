\description{"Run a k6 load test and capture the summary."}

\image{"grafana/k6:1.7.1"}

Run \input{script, required} as a k6 test script. The script is
written to a file and passed to `k6 run`. All extra_inputs declared
on the step are available as named files in `./context/` for the
script to reference (e.g. test data, configuration).

\input{options} provides additional k6 CLI flags as a single string
(e.g. `--vus 10 --duration 30s`). When omitted, k6 uses its
defaults or whatever `export let options = {...}` the script declares.

\output{exitCode, schema={"type":"integer"}} records k6's exit code.
Non-zero means at least one threshold failed or the run errored.

\output{summary, schema={"type":"object"}} captures the JSON summary
produced by `--summary-export`. Contains metrics, thresholds, and
check results. Downstream steps can branch on specific metric values.

Stdout and stderr (k6's progress output) flow through the runner's
log pipeline.

```sh
set +e
cat ./context/script > $TMPDIR/test.js

opts=""
[ -f ./context/options ] && opts="$(cat ./context/options)"

k6 run $opts --summary-export=./outputs/summary.json $TMPDIR/test.js
ec=$?

printf '%d' "$ec" > ./outputs/exitCode
exit "$ec"
```
