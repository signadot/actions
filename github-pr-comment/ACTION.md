\description{"Post a comment on a GitHub pull request."}

Post a markdown comment on pull request \input{pr, required} in
repository \input{repo, required} (format: `owner/name`).

\input{body, required} is the comment body. Supports full GitHub
markdown — code fences, tables, task lists, etc. When wiring from
an upstream step, use the `eval` action to format the body if
needed.

\input{token, required} is a GitHub personal access token or
GitHub App token with `repo` scope (or `issues:write` for
fine-grained tokens). Wire via `--secret` at execution time so
the token never appears in plan outputs.

\output{statusCode, schema={"type":"integer"}} records the HTTP
status. 201 means the comment was created successfully.

\output{response, schema={"type":"object"}} captures the GitHub
API response JSON (includes the comment URL, ID, etc.).

```sh
set +e
repo="$(cat ./context/repo)"
pr="$(cat ./context/pr)"
token="$(cat ./context/token)"

# Use jq to safely encode the body as a JSON string value,
# handling all special characters (newlines, quotes, backslashes).
payload=$(jq -n --rawfile body ./context/body '{"body": $body}')

status=$(curl -sS -o ./outputs/response.json -w "%{http_code}" \
  -X POST \
  -H "Authorization: Bearer $token" \
  -H "Content-Type: application/json" \
  -H "Accept: application/vnd.github+json" \
  -d "$payload" \
  "https://api.github.com/repos/$repo/issues/$pr/comments")

printf '%d' "$status" > ./outputs/statusCode
exit 0
```
