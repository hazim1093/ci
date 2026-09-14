# ci

Shared, reusable GitHub Actions workflows for hazim1093's repositories. Public by design — nothing repo-specific or private lives here; callers pass their own paths via `inputs`.

Version pins (kustomize, kubeconform, kubernetes) live here, so tool bumps happen in one place.

Each workflow is standalone: independent triggers, no `needs:` coupling, usable alone in any repo — k8s repos, app repos, docs repos. When a repo calls both, they run as two parallel checks.

## Caller permissions

A called workflow's jobs can only request permissions the **caller** explicitly grants. With no `permissions:` block in the caller, every scope is `none`, and GitHub rejects the run before it starts:

```
Invalid workflow file: .github/workflows/secret-scan.yml#L10
The nested job 'secret-scan' is requesting 'pull-requests: write',
but is only allowed 'pull-requests: none'.
```

So every caller must declare the scopes its called workflow needs:

| Workflow | Caller `permissions:` |
| --- | --- |
| `k8s-validation` | `contents: read` |
| `secret-scan` | `contents: read`, `pull-requests: write` |

## Reusable workflows

### k8s-validation

Kustomize-builds and kubeconform-validates every changed manifest unit directory under the given roots (Flux postBuild placeholders are substituted with sample values first).

Callers pass the manifest roots they manage, space-separated:

```yaml
permissions:
  contents: read

jobs:
  validate:
    uses: hazim1093/ci/.github/workflows/k8s-validation.yml@main
    with:
      roots: kubernetes/components
```

For multiple roots:

```yaml
    with:
      roots: kubernetes/apps kubernetes/components
```

Typical caller triggers (watch only the paths you manage):

```yaml
on:
  pull_request:
    paths:
      - 'kubernetes/components/**'
  push:
    branches: [main]
    paths:
      - 'kubernetes/components/**'
```

### secret-scan

Gitleaks scan of every push/PR; comments the offending files on the PR, fails the check when leaks are found, warns on main.

```yaml
permissions:
  contents: read
  pull-requests: write

jobs:
  secret-scan:
    uses: hazim1093/ci/.github/workflows/secret-scan.yml@main
```

No inputs. The caller keeps its own `on:` triggers:

```yaml
on:
  pull_request:
  push:
    branches: [main]
```

## Pinning

Callers reference `@main`. There is no release or tag step: the workflows are consumed as-is, so a merge here lands in every caller on its next run.

To raise the tool versions, bump `KUSTOMIZE_VERSION` / `KUBECONFORM_VERSION` / `KUBERNETES_VERSION` in the workflow `env:` block — every caller picks it up with no change on their side.
