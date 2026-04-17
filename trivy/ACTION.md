\description{"Scan Kubernetes manifests or filesystems for vulnerabilities and misconfigurations."}

\image{"aquasec/trivy:0.70.0"}

Scan \input{manifest, required} for vulnerabilities and
misconfigurations using Trivy. The manifest should be a YAML
document containing Kubernetes resource definitions. Trivy checks
both the resource configuration (misconfigurations) and any
container images referenced in the manifest (known CVEs).

\input{severity} filters results by severity level (e.g.
`CRITICAL,HIGH`). When omitted, all severities are reported.

\input{options} provides additional Trivy CLI flags as a single
string (e.g. `--ignore-unfixed`). When omitted, defaults apply.

\output{exitCode, schema={"type":"integer"}} records Trivy's exit
code: 0 = no issues found, 1 = issues detected.

\output{report, schema={"type":"object"}} captures the JSON scan
report. Contains vulnerabilities grouped by resource and severity.
Downstream steps can branch on specific CVE IDs or severity counts.

Stdout and stderr flow through the runner's log pipeline.

```sh
set +e
cat ./context/manifest > /tmp/manifest.yaml

opts=""
[ -f ./context/severity ] && opts="$opts --severity $(cat ./context/severity)"
[ -f ./context/options ] && opts="$opts $(cat ./context/options)"

trivy config $opts --format json --output ./outputs/report.json /tmp/manifest.yaml
ec=$?

printf '%d' "$ec" > ./outputs/exitCode
exit "$ec"
```
