\description{"Run a k6 load test and capture the summary."}

\image{"grafana/k6:1.7.1"}

Run \input{script, required} as a k6 test script. The script is placed
in the action's working directory and invoked via `k6 run`. All
extra_inputs declared on the step are available as named files in
`./context/` for the script to reference (e.g. test data, configuration)
— `open('./context/<name>')` from inside the script resolves there
because k6's `open()` is relative to the script's directory.

\input{target_url} sets the `TARGET_URL` environment variable in the
script's runtime. The script reads it via `__ENV.TARGET_URL`. The
runner executes in-cluster, so wire `target_url` to the in-cluster
`.svc` address (e.g. `http://<svc>.<namespace>.svc:<port>`) to point
a k6 plan at a sandboxed environment without composing the URL
inside the script.

\input{options} provides additional k6 CLI flags as a single string
(e.g. `--vus 10 --duration 30s`). When omitted, k6 uses its
defaults or whatever `export let options = {...}` the script declares.

\output{exit_code, schema={"type":"integer"}} records k6's exit code.
Non-zero means at least one threshold failed or the run errored. The
action's bash exits with this code, so a threshold violation fails
the step (and propagates to the plan). Contrast with the `playwright`
action, which always exits 0 and surfaces pass/fail only via its
`exit_code` output — wire downstream `check` steps accordingly when
mixing the two.

\output{summary, schema={"type":"object"}} captures the JSON summary
produced by `--summary-export`. Contains metrics, thresholds, and
check results. Downstream steps can branch on specific metric values.
Common key paths: `summary.metrics.http_req_duration['p(95)']`
(latency percentile), `summary.metrics.http_req_failed.value`
(failure rate `0.0`–`1.0`), `summary.metrics.http_reqs.count`
(request count).

**Two k6 fields read the opposite of their natural names** — read
both carefully when authoring assertions:

1. `thresholds.<expr>` boolean is `true` when the threshold was
   *violated*, `false` when it passed.
2. On Rate metrics, `.passes` counts the underlying check evaluating
   *true* and `.fails` counts it evaluating *false*. For
   `http_req_failed` (the underlying check is "did this request
   fail?") that means `.fails` is the count of *successful*
   requests, not failures. Use `.value` (the rate, `0.0`–`1.0`)
   when you want the failure rate.

The `OUTPUTS_DIR` environment variable points at the action's output
directory. Test code can write arbitrary files there (custom data
dumps from `handleSummary`, HAR exports, etc.) and the plan author
declares matching `extra_outputs` on the step to capture them. See
*Capturing custom artifacts* under Authoring rules below.

Stdout and stderr (k6's progress output) flow through the runner's
log pipeline.

## Authoring rules

**Routing through a sandbox.** When the step has `routingContext`
set, the runner exposes `SIGNADOT_ROUTING_KEY` to the script
(`__ENV.SIGNADOT_ROUTING_KEY`). Inject the cluster's routing-key
headers on every outbound request. **Discover what your cluster
accepts via the `signadot-plan` skill before authoring** — the
cluster's `customHeaders` may include names beyond the
always-accepted `baggage`/`tracestate` pair, and missing them
silently routes to baseline. The snippet below shows the
always-accepted pair only; treat it as a starting point, not a
complete header set:

```js
import http from 'k6/http';
const KEY = __ENV.SIGNADOT_ROUTING_KEY;
const headers = {
  baggage: `sd-routing-key=${KEY}`,
  tracestate: `sd-routing-key=${KEY}`,
  // Also inject any clusterConfig.routing.customHeaders — see signadot-plan skill
};

export default function () {
  http.get(__ENV.TARGET_URL, { headers });
}
```

For suites with many requests, attach the headers via the `params`
default at module scope, or pass per-request via the `params`
argument to each `http.<method>` call.

**Asserting on a metric — two patterns, different effects:**

- **k6 `thresholds`** (in the script) — *plan-level gate.* A
  violation makes k6 exit non-zero, failing the step and the plan.
  The plan's `status.phase` reflects the assertion outcome:

  ```js
  export const options = {
    thresholds: {
      http_req_duration: ['p(95)<500'],
      http_req_failed:   ['rate<0.01'],
    },
  };
  ```

- **Downstream `check` step** — *assertion record, not a gate.* The
  `check` action exits 0 whether the expression is true or false
  (by design — the outcome lives in `result.error`). The plan's
  phase stays *succeeded* on a violation; downstream consumers
  inspect check results to decide overall pass/fail. Use this when
  assertions are part of a structured report (CI output, plan-level
  outputs) rather than driving plan-level pass/fail. Also useful
  when the assertion needs an expression k6's threshold DSL can't
  express, or composes inputs from multiple steps.

If you need the plan's phase to reflect the assertion, use k6
thresholds (or wire a downstream shell/eval step that reads the
check result and exits non-zero on failure).

**Always include `http_req_failed`** as one of your gates —
regardless of which form. Without it, a connection-refused or
all-non-2xx run reports `"16033 complete iterations"` with
`data_received: 0 B` and zero durations; k6 won't surface the
failure in its summary unless a threshold or `check()` catches it.
**Pattern: zero metrics + nonzero iteration count = HTTP non-2xx or
connection failure.** Most common silent-success-shaped failure mode.

**Capturing custom artifacts.** Test code writes to
`${__ENV.OUTPUTS_DIR}/<name>`; the plan author declares the matching
`extra_outputs` on the step. Two constraints:

1. Output names follow the runner's identifier rule
   (`^[a-zA-Z_][a-zA-Z0-9_]*$`) — alphanumeric + underscore, no dots
   or dashes.
2. The file name on disk must be exactly the output name **without
   extension**. Set `metadata.contentType` on the `extra_output` so
   downstream consumers render the file correctly.

The k6-idiomatic place for custom dumps is `handleSummary`, which
runs once after all VUs finish:

```yaml
- id: load
  action: { actionID: ... }
  args:
    values:
      script: |
        import http from 'k6/http';
        export default function () {
          http.get(__ENV.TARGET_URL);
        }
        export function handleSummary(data) {
          return {
            [`${__ENV.OUTPUTS_DIR}/perf_dump`]: JSON.stringify(data),
          };
        }
  extraOutputs:
  - name: perf_dump
    metadata: { contentType: application/json }
```

For built-in CLI outputs (CSV time-series, JSON metrics streams),
pass the path through \input\{options} as a workdir-relative path —
k6 runs from the workdir, and `./outputs/<name>` is what the runner
reads as an extra_output:

```yaml
options: --out csv=outputs/timeseries
extraOutputs:
- name: timeseries
  metadata: { contentType: text/csv }
```

```sh
set +e
cat ./context/script > ./test.js

opts=""
[ -f ./context/options ] && opts="$(cat ./context/options)"
[ -f ./context/target_url ] && export TARGET_URL="$(cat ./context/target_url)"

export OUTPUTS_DIR="$PWD/outputs"

k6 run $opts --summary-export=./outputs/summary.json ./test.js
ec=$?

printf '%d' "$ec" > ./outputs/exit_code
exit "$ec"
```
