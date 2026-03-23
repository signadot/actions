\requires{"actionbox"}

Execute an HTTP request against \input{url, required} and capture the full
roundtrip as structured JSON written to \output{capture}.

\input{method, default="GET"} specifies the HTTP method.
\input{headers, schemaRef="schemas/headers.json"} is a list of `Key: Value` headers to include in the request.
\input{body} is the request body.
\input{timeout, default="30s"} sets the request timeout in Go duration format.
\input{follow_redirects, default="true"} controls whether HTTP redirects are followed.
\input{insecure, default="false"} skips TLS certificate verification when set.

**Routing key injection:** If the `SIGNADOT_ROUTING_KEY` environment variable is
set, routing headers are automatically injected into the request using multiple
propagation formats (OTel baggage, tracestate, Jaeger, Datadog, and custom
headers from cluster config).

**Header sanitization:** Sensitive headers are removed from the capture output.
Request: `Cookie`, `Authorization`, `Authentication`. Response: `Set-Cookie`.
Additional headers can be configured via cluster config.

**Body parsing:**

- `application/json` — parsed as structured JSON
- `application/x-www-form-urlencoded` — parsed as `map[string][]string`
- Other content types or parse errors — base64-encoded string
- Empty body — `null`

Response bodies are capped at 200KB.

**Exit codes:**

- `0` — request completed (inspect the capture JSON for status/error details)
- `1` — usage error (bad flags, file read error, invalid header format)

**Output format:**

\output{capture, schemaRef="schemas/capture.json"} is a JSON object:

```json
{
  "request": {
    "proto": "HTTP/1.1",
    "method": "GET",
    "uri": "/api/items?page=1",
    "host": "example.com",
    "query": {"page": "1"},
    "message": {
      "startedAt": "2026-03-16T12:34:56Z",
      "headers": {"Accept": "application/json"},
      "body": null
    }
  },
  "response": {
    "statusCode": 200,
    "proto": "HTTP/1.1",
    "message": {
      "startedAt": "2026-03-16T12:34:57Z",
      "finishedAt": "2026-03-16T12:34:57.005Z",
      "headers": {"Content-Type": "application/json"},
      "body": {"items": [{"name": "a"}], "total": 1}
    }
  }
}
```

On transport error, the response contains an `error` field instead:

```json
{
  "request": { "..." : "..." },
  "response": {
    "error": "dial tcp: connection refused"
  }
}
```

```sh
args=(--url "$(cat ./input/url)")

[ -f ./input/method ] && args+=(--method "$(cat ./input/method)")
[ -f ./input/timeout ] && args+=(--timeout "$(cat ./input/timeout)")
[ -f ./input/follow_redirects ] && args+=(--follow-redirects="$(cat ./input/follow_redirects)")
[ -f ./input/insecure ] && args+=(--insecure="$(cat ./input/insecure)")
[ -f ./input/body ] && args+=(--body-file ./input/body)

if [ -f ./input/headers ]; then
  while IFS= read -r header; do
    args+=(--header "$header")
  done < ./input/headers
fi

actionbox request-http "${args[@]}" > ./output/capture
```
