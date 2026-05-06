\description{"Run Playwright browser tests and capture results."}

\image{"mcr.microsoft.com/playwright:v1.59.1"}

Run \input{script, required} as a Playwright test. The script should
be a JavaScript or TypeScript file that uses the Playwright test API.
All extra_inputs declared on the step are available as named files in
`./context/` — use them for test data, configuration, or URL targets.

\input{options} provides additional Playwright CLI flags as a single
string (e.g. `--retries=2 --workers=1`). The action runs without your
repo's `playwright.config.ts`, so flags that depend on a config —
notably `--project=<name>` — silently no-op or error unless the test
script defines projects inline via `test.use({ ... })`. Standalone
flags (`--retries`, `--workers`, `--timeout`, `--max-failures`,
`--grep`) work fine. **Do not pass `--reporter`** — the action already
wires `--reporter=list,json` to produce the `report` output, and a
later `--reporter=...` would replace that, silently breaking the
JSON report. When omitted, Playwright runs the test as-is with its
defaults.

\input{base_url} sets the `BASE_URL` environment variable in the
script's runtime. Playwright does *not* auto-wire env vars into its
config — your test code has to read `process.env.BASE_URL` explicitly,
either by passing it to `page.goto()`
(`page.goto(process.env.BASE_URL + '/login')`) or by registering it
once at the top of the script via
`test.use({ baseURL: process.env.BASE_URL })` so bare `page.goto('/')`
resolves against it. Three viable targets, pick by what the test
depends on:

- **In-cluster `.svc` address** (e.g.
  `http://<svc>.<namespace>.svc:<port>`) — fastest path; the runner
  is in-cluster, the request stays in-cluster. Right when the test
  just exercises HTTP/JSON endpoints, doesn't load real frontend
  assets, and doesn't depend on TLS, hostname, or cookie domain.
- **Sandbox preview URL** (`*.preview.signadot.com`) — gives real
  TLS, but on a Signadot-owned hostname. Works for many
  full-frontend tests, but breaks anything pinned to the production
  domain: SSO / OAuth flows whose redirect URIs only accept the
  production host, cookies bound by `Domain=app.example.com`,
  production CSP rules, etc.
- **Production / external domain + routing key header** — set
  `base_url` to the real production URL (e.g.
  `https://app.example.com`) and inject the routing key
  (typically `baggage: sd-routing-key=$SIGNADOT_ROUTING_KEY`) into
  every outbound request via `test.use({ extraHTTPHeaders: { ... } })`
  or a `page.route('**/*', ...)` hook. The cluster's edge reads the
  header and routes to the sandbox. Right when the test exercises
  auth flows pinned to the production domain. Requires the cluster
  to actually serve traffic for that hostname (production cluster
  or production-shaped staging) and `routingContext` set on the
  step so `SIGNADOT_ROUTING_KEY` is in env.

Default to `.svc` for cheap API-shape tests; escalate to the preview
URL when bare `.svc` trips a TLS / host issue; escalate to the
production domain when the preview URL trips a domain-pinned auth /
cookie / CSP issue.

\input{dependencies} is a space-separated list of additional npm
package specs to install before the test runs (e.g.,
`@playwright/experimental-ct-react@1.59.1 axios@^1.6`). When omitted,
only `@playwright/test` is installed. `@playwright/test` itself is
always installed and pinned to the image's playwright version — it is
not overridable through this input; fork the action and change the
declared image to use a different Playwright version.

\input{capture_artifacts, default="false"} controls whether the
action bundles Playwright's per-test `./test-results/` directory
after the run. Set to the string `"true"` to enable (literally the
strings `"true"` / `"false"` — this input has no schema, so the
script reads its raw text). The action handles the bundling in its
post-run bash. See *Capturing trace, video, and other Playwright
artifacts* under Authoring rules below for which outputs are
surfaced.

\output{test_artifacts, contentType="application/gzip"}
is a tarball of `./test-results/`. Produced when
`capture_artifacts="true"` and `./test-results/` exists. Includes
whatever Playwright wrote into that directory: `trace.zip`,
`video.webm`, automatic failure screenshots (`test-failed-N.png`
when `screenshot: 'on'` or `'only-on-failure'`), visual-regression
diffs (`<name>-expected.png` / `-actual.png` / `-diff.png` from
failed `toHaveScreenshot()` assertions), HAR files (when `recordHar`
is configured), `error-context.md` on failure (Playwright 1.50+),
and any custom attachments written via `testInfo.attach()`. Does
*not* include the HTML reporter's `./playwright-report/` directory
or the blob reporter's `./blob-report/` — those are separate paths
the action doesn't capture.

\output{trace, contentType="application/zip"} is a
standalone Playwright trace. Produced only when *exactly one*
trace.zip exists in `./test-results/` (typical of single-focal-test
plans); the dashboard renders Playwright's trace format inline.
Plans that ref `steps.X.outputs.trace` against a multi-test run
will see it missing — use `test_artifacts` for that case.

\output{video, contentType="video/webm"} is a
standalone test recording. Produced only when *exactly one* `.webm`
exists in `./test-results/`; the dashboard renders inline. Same
multi-test caveat as `trace`.

\output{exit_code, schema={"type":"integer"}} records Playwright's
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
   extension**. Playwright infers image type from the path's extension
   and rejects extension-less paths (`unsupported mime type "null"`),
   so set `type: 'png'` (or `'jpeg'`) explicitly on `page.screenshot`.

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
          await page.screenshot({
            path: `${process.env.OUTPUTS_DIR}/login_screen`,
            type: 'png',
          });
        });
  extraOutputs:
  - name: login_screen
    metadata: { contentType: image/png }
```

**Porting from a config-based project (`playwright.config.ts`).** The
action runs without your repo's `playwright.config.ts`. Anything that
normally lives there has to move into the test script via
`test.use({ ... })` at the top, or it doesn't take effect. The most
common items, with their inline equivalents:

| Lives in `playwright.config.ts` | Inline equivalent in the script |
|---|---|
| `use.baseURL` | `test.use({ baseURL: process.env.BASE_URL })` |
| `use.trace` / `use.video` | `test.use({ trace: 'on', video: 'on' })` |
| `use.extraHTTPHeaders` | `test.use({ extraHTTPHeaders: { ... } })` |
| `use.viewport`, `use.locale`, `use.timezoneId`, etc. | Same key under `test.use({ ... })` |
| `projects: [{ name: ..., use: ... }]` | Define inline; `--project=<name>` won't resolve against a missing config |
| `timeout`, `expect.timeout` | `test.use({ timeout: ... })` for the test timeout; `expect.configure({ timeout: ... })` once at file scope, or per-call options on assertions, for the expect timeout |
| `reporter` | **Cannot override** — the action's `--reporter=list,json` is required to produce the `report` output. Playwright keeps only the last `--reporter` flag, so a custom reporter passed via `\input\{options}` would silently break the JSON report. For raw per-test artifacts (screenshots, traces, videos, HAR, attachments), use `test_artifacts` instead of swapping reporters. |

Things that *cannot* be ported inline: config-defined fixtures, global
setup/teardown files, custom test runners. If the suite depends on
those, package the suite (and its config) as a `\image\{...}`-based
custom action instead of running it through this one.

**Capturing trace, video, and other Playwright artifacts.** Set
`capture_artifacts: "true"` on the step and enable trace/video in
`test.use({ trace: 'on', video: 'on' })`. The action's post-run bash
bundles the artifacts after `npx playwright test` exits — strictly
after every worker has flushed its trace.zip and video.webm.

The three outputs the action declares for these (`test_artifacts`,
`trace`, `video` — see top-of-file) are produced conditionally:

- `test_artifacts` is produced whenever `./test-results/` exists.
  Tarball of the full per-test layout; extract with `tar xzf`.
- `trace` and `video` are standalone artifacts produced *only* when
  exactly one of each exists across the run — the bash doesn't pick
  a winner when multiple tests produce traces or videos.

Pick by use case:

- *Single focal test* (smoke check, regression repro, single user
  flow) typically produces one trace and one video. Refs to
  `steps.X.outputs.trace` and `steps.X.outputs.video` resolve, and
  the dashboard renders both inline.
- *Multi-test suite* produces N of each. Use
  `steps.X.outputs.test_artifacts` to get the full tarball;
  `trace` / `video` are absent for the run.

```yaml
- id: tests
  action: { actionID: ... }
  args:
    values:
      script: |
        const { test } = require('@playwright/test');
        test.use({ trace: 'on', video: 'on' });
        test('login flow', async ({ page }) => {
          await page.goto(process.env.BASE_URL + '/login');
          // ...assertions
        });
      capture_artifacts: "true"
```

```sh
set +e
mkdir -p "$TMPDIR/pw"
cat ./context/script > "$TMPDIR/pw/test.spec.js"

opts=""
[ -f ./context/options ] && opts="$(cat ./context/options)"
[ -f ./context/base_url ] && export BASE_URL="$(cat ./context/base_url)"
extra_deps=""
[ -f ./context/dependencies ] && extra_deps="$(cat ./context/dependencies)"
capture_artifacts="false"
[ -f ./context/capture_artifacts ] && capture_artifacts="$(cat ./context/capture_artifacts)"

outdir="$PWD/outputs"
export OUTPUTS_DIR="$outdir"
cd "$TMPDIR/pw"

# The mcr.microsoft.com/playwright image ships browsers + node + npm
# but not @playwright/test (Microsoft expects users to bring their own
# project). Bootstrap a minimal one here so `require('@playwright/test')`
# in the user's script resolves. Browsers come from /ms-playwright via
# PLAYWRIGHT_BROWSERS_PATH set in the image — no browser download.
echo '{"name":"plan-test","version":"1.0.0","private":true}' > package.json
npm install --no-save --no-audit --no-fund --silent \
  @playwright/test@1.59.1 \
  $extra_deps

PLAYWRIGHT_JSON_OUTPUT_NAME="$outdir/report.json" \
  npx playwright test test.spec.js \
    --reporter=list,json \
    $opts
ec=$?

# Bundle Playwright artifacts when the plan opts in. Runs strictly
# after npx playwright test (and all its workers) exit, which is
# strictly after every browser context has closed and flushed its
# trace.zip / video.webm — no in-script race. The action's declared
# outputs (test_artifacts, trace, video) are produced conditionally;
# plan authors ref them via steps.X.outputs.<name> as needed.
if [ "$capture_artifacts" = "true" ] && [ -d ./test-results ]; then
    # Bundle the full per-test layout. Don't swallow tar errors: a
    # partial archive would mislead downstream steps. On failure we
    # remove the (possibly truncated) tarball and let stderr surface.
    if ! tar czf "$outdir/test_artifacts" ./test-results; then
        rm -f "$outdir/test_artifacts"
    fi

    # Surface a standalone trace.zip iff exactly one per-test trace
    # exists. trace.zip is the Playwright-produced literal name;
    # filter exactly to skip worker scratch artifacts.
    traces=$(find ./test-results -name 'trace.zip' 2>/dev/null)
    if [ -n "$traces" ] && [ "$(echo "$traces" | wc -l)" = "1" ]; then
        cp "$traces" "$outdir/trace"
    fi
    # Same for video.webm. Don't use *.webm — Playwright also writes
    # worker-scratch page@<hash>.webm files in .playwright-artifacts-N/
    # that aren't per-test recordings.
    videos=$(find ./test-results -name 'video.webm' 2>/dev/null)
    if [ -n "$videos" ] && [ "$(echo "$videos" | wc -l)" = "1" ]; then
        cp "$videos" "$outdir/video"
    fi
fi

printf '%d' "$ec" > "$outdir/exit_code"
exit 0
```
