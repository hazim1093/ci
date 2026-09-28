# Local CRD schema overrides

kubeconform (in `k8s-validation.yml`) checks this directory before falling
back to the [CRDs-catalog](https://github.com/datreeio/CRDs-catalog) mirror.
Files here are copies of an upstream CRD's schema with a specific, documented
correction — never a hand-authored schema, and never a blanket loosening.

## grafana.integreatly.org/grafanaalertrulegroup_v1beta1.json

Upstream (grafana-operator) marks `spec.interval` and
`spec.rules[].keepFiringFor` as `format: duration`, which JSON Schema defines
as an RFC 3339 / ISO 8601 duration (e.g. `PT1M`). Both fields are actually
`metav1.Duration` and parsed with Go's `time.ParseDuration` (e.g. `1m`), so
any value that's actually valid for the field fails the `format` assertion.

This copy removes the incorrect `format` keyword from those two fields only.
The `pattern` regex on each (which does correctly constrain Go-style
durations) is left untouched, so validation strength is otherwise identical
to the upstream schema.

To refresh after an upstream schema change: re-fetch from CRDs-catalog and
re-apply the same two deletions, or re-run:

```
python3 - <<'EOF'
import json
path = "grafanaalertrulegroup_v1beta1.json"
data = json.load(open(path))
for prop in (data["properties"]["spec"]["properties"]["interval"],
             data["properties"]["spec"]["properties"]["rules"]["items"]["properties"]["keepFiringFor"]):
    prop.pop("format", None)
json.dump(data, open(path, "w"), indent=2)
open(path, "a").write("\n")
EOF
```
