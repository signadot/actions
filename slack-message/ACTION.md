\description{"Post a message to a Slack channel via incoming webhook."}

Send a message to the Slack channel configured on
\input{webhook_url, required}. The webhook URL embeds the
authentication — no separate token needed. Create one at
https://api.slack.com/messaging/webhooks.

\input{text, required} is the message text. Supports Slack's
mrkdwn format — `*bold*`, `_italic_`, `` `code` ``, `>quote`,
and `\n` for newlines. For rich layouts (sections, fields,
buttons), build a blocks JSON payload with the `eval` action
upstream and pass it as the text.

Wire `webhook_url` via `--secret` at execution time so it never
appears in plan outputs.

\output{statusCode, schema={"type":"integer"}} records the HTTP
status. 200 means the message was accepted.

\output{response} captures the Slack response body ("ok" on
success, error string on failure).

```sh
set +e
webhook_url="$(cat ./context/webhook_url)"

payload=$(jq -n --rawfile text ./context/text '{"text": $text}')

status=$(curl -sS -o ./outputs/response -w "%{http_code}" \
  -X POST \
  -H "Content-Type: application/json" \
  -d "$payload" \
  "$webhook_url")

printf '%d' "$status" > ./outputs/statusCode
exit 0
```
