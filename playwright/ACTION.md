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

The `OUTPUTS_DIR` environment variable points at the action's output
directory. Test code can write arbitrary files there (screenshots,
traces, HAR files) and the plan author declares matching
`extra_outputs` on the step to capture them. See *Capturing
screenshots and other binary artifacts* under Authoring rules below.

Stdout and stderr (Playwright's progress output) flow through the
runner's log pipeline.

## Authoring rules

**Capturing screenshots and other binary artifacts.** Test code writes
to `${OUTPUTS_DIR}/<name>`; the plan author declares the matching
`extra_outputs` on the step. Two constraints:

1. Output names follow the runner's identifier rule
   (`^[a-zA-Z_][a-zA-Z0-9_]*$`) — alphanumeric + underscore, no dots
   or dashes. Pick `login_screen`, not `login.png` or `login-screen`.
2. The file name on disk must be exactly the output name **without
   extension**. Playwright accepts extension-less paths and writes the
   bytes for you (PNG by default for `page.screenshot`).

Set `metadata.contentType` on the `extra_output` so downstream
consumers render the file correctly:

```yaml
- id: tests
  action: { actionID: ... }
  args:
    values:
      script: |
        const { test } = require('@playwright/test');
        test('login', async ({ page }) => {
          await page.goto(process.env.BASE_URL + '/login');
          await page.screenshot({ path: `${process.env.OUTPUTS_DIR}/login_screen` });
        });
  extraOutputs:
  - name: login_screen
    metadata: { contentType: image/png }
```

```sh
set +e
mkdir -p "$TMPDIR/pw"
cat ./context/script > "$TMPDIR/pw/test.spec.js"

opts=""
[ -f ./context/options ] && opts="$(cat ./context/options)"
[ -f ./context/baseURL ] && export BASE_URL="$(cat ./context/baseURL)"

outdir="$PWD/outputs"
export OUTPUTS_DIR="$outdir"
cd "$TMPDIR/pw"

# The mcr.microsoft.com/playwright image ships browsers + node + npm
# but not @playwright/test (Microsoft expects users to bring their own
# project). Bootstrap a minimal one here so `require('@playwright/test')`
# in the user's script resolves. Browsers come from /ms-playwright via
# PLAYWRIGHT_BROWSERS_PATH set in the image — no browser download.
echo '{"name":"plan-test","version":"1.0.0","private":true}' > package.json
npm install --no-save --no-audit --no-fund --silent @playwright/test@1.59.1

PLAYWRIGHT_JSON_OUTPUT_NAME="$outdir/report.json" \
  npx playwright test test.spec.js \
    --reporter=list,json \
    $opts
ec=$?

printf '%d' "$ec" > "$outdir/exitCode"
exit 0
```
