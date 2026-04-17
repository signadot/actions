\description{"Run Playwright browser tests and capture results."}

\image{"mcr.microsoft.com/playwright:v1.59.1"}

Run \input{script, required} as a Playwright test. The script should
be a JavaScript or TypeScript file that uses the Playwright test API.
All extra_inputs declared on the step are available as named files in
`./context/` — use them for test data, configuration, or URL targets.

\input{options} provides additional Playwright CLI flags as a single
string (e.g. `--project=chromium --retries=2`). When omitted,
Playwright runs all projects defined in the test or its defaults.

\input{baseURL} sets the `BASE_URL` environment variable. Playwright
tests that use `page.goto('/')` resolve against this — wire it to
the sandbox's preview URL for end-to-end testing against a sandboxed
environment.

\output{exitCode, schema={"type":"integer"}} records Playwright's
exit code: 0 = all tests passed, 1 = failures.

\output{report, schema={"type":"object"}} captures the JSON test
report. Contains per-test pass/fail status, durations, and error
messages. Downstream steps can branch on failure counts or specific
test names.

Stdout and stderr (Playwright's progress output) flow through the
runner's log pipeline.

```sh
set +e
mkdir -p "$TMPDIR/pw"
cat ./context/script > "$TMPDIR/pw/test.spec.js"

opts=""
[ -f ./context/options ] && opts="$(cat ./context/options)"
[ -f ./context/baseURL ] && export BASE_URL="$(cat ./context/baseURL)"

cd "$TMPDIR/pw"
npx playwright test test.spec.js \
  --reporter=json \
  $opts \
  > ./playwright-report.json 2>&1
ec=$?

cp "$TMPDIR/pw/playwright-report.json" ./outputs/report.json 2>/dev/null
printf '%d' "$ec" > ./outputs/exitCode
exit 0
```
