\description{"Aggregate the verdicts of multiple check steps into one result with a pass/fail summary."}
\requires{"actionbox"}
\extra_inputs_schema{default={}}

Collect the `result` outputs of one or more `check` steps — passed as
extra_inputs — into a single \output{results, schemaRef="schemas/results.json"}.
Wire each check's `result` into this action via `refs`; the input names don't
matter, every input is read as a verdict.

This is the aggregation point for a batch of checks: the checks run
independently (in parallel), each records its own verdict, and this action
reduces them to one object carrying every verdict plus `total` / `failed` counts.

**This action always completes — it does not fail the run on a failed check.** A
failed assertion is a *result*, not an execution error, so the verdict lives in
the report and the **pass/fail decision is made by the consumer** (e.g. CI reads
`failed` and fails the build). This mirrors how a Smart Test job completes and
reports its verdict separately, and it keeps `results` a normal, fully-written
output of a *successful* step — so it is always collectable, including on the
failing-suite case.

**Output.** `results` is a single JSON object — a summary plus the verdicts:

```json
{ "total": 135, "failed": 1, "checks": [ { "check": "...", "error"?: { "message": "...", "attrs"?: {...} } }, ... ] }
```

Read `failed` / `total` directly to gate. A one-line `passed/failed` summary is
also written to the step log (stderr).

**Exit codes:**

- `0` — completed; the report was written (regardless of how many checks failed)
- non-zero — a usage or input error (missing `--context-dir`, or an unreadable /
  malformed verdict); these are genuine execution errors and fail the step

## Authoring rules

Wire each `check` step's `result` into this action as an `extra_input` via
`refs` — the input names are arbitrary; every input is treated as a verdict. The
action always succeeds; the pass/fail decision belongs to the consumer, which
reads `failed` from `results` (e.g. in CI: `signadot plan x get-output <exec>
results | jq '.failed'`).

**Example:**

```yaml
- id: results
  action: { actionID: check-aggregate }
  extraInputs:
    - { name: ck1 }
    - { name: ck2 }
  args:
    refs:
      ck1: steps.check1.outputs.result
      ck2: steps.check2.outputs.result
output:
  results: steps.results.outputs.results
```

```sh
actionbox check-aggregate --context-dir ./context > ./outputs/results.json
```
