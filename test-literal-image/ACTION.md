\description{"Test action with a fixed alpine image."}

\image{"alpine:3.19"}

\output{msg}

```sh
set +e
. /etc/os-release
printf 'hello from %s:%s' "$ID" "$VERSION_ID" > ./outputs/msg
```
