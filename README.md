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
| `docker-build-push` | `contents: read`, `packages: write` |
| `k8s-preview` | `contents: read`, `packages: write`, `pull-requests: write`, `issues: write`, `checks: write` |
| `k8s-preview-teardown` | `contents: read`, `pull-requests: write`, `issues: write` |

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

### docker-build-push

Builds one image and pushes it to `ghcr.io/<caller owner>/<image>` (`contents: read`,
`packages: write` in the caller). One call per image, so a two-image app runs two of these
in parallel; each gets its own `gha` cache scope keyed by image name.

| Input | Meaning |
| --- | --- |
| `image` | Image name without registry/tag, e.g. `screener-api` |
| `dockerfile` | Dockerfile path, relative to `context` |
| `context` | Build context (default `.`) |
| `version` | Image tag; empty = the caller's latest release tag, else `latest` |
| `tag_as_latest` | Also push `:latest` |
| `build_args` | Newline-separated `KEY=VALUE` build arguments |

Caller (keeps `workflow_dispatch` so it can also be run by hand):

```yaml
on:
  workflow_dispatch:
    inputs:
      version: { description: "Image tag", required: false, default: "" }
      tag_as_latest: { type: boolean, default: false }
  workflow_call:
    inputs:
      version: { type: string, required: false, default: "" }
      tag_as_latest: { type: boolean, default: false }

permissions:
  contents: read
  packages: write

jobs:
  publish:
    uses: hazim1093/ci/.github/workflows/docker-build-push.yml@main
    with:
      image: my-app
      dockerfile: Dockerfile
      version: ${{ inputs.version }}
      tag_as_latest: ${{ inputs.tag_as_latest }}
```

### k8s-preview / k8s-preview-teardown

Ephemeral per-PR environments on the home-lab cluster: build `pr-<n>` images, copy the app's
secrets into a fresh namespace, apply a kustomize overlay from a private manifests repo, wait
for rollout, then comment on the PR. Teardown deletes namespace + ReferenceGrant on PR close
or `/teardown-preview`. Callers keep their own `on:` triggers and pass `secrets: inherit`
(the workflows declare `LOCAL_DOMAIN`, `GH_APP_ID`, `GH_APP_PRIVATE_KEY`).

Conventions derived from `app`:

| Thing | Value |
| --- | --- |
| Preview namespace | `pr-<app>-<PR>` |
| ReferenceGrant (keda-http ns) | `pr-<app>-<PR>-to-keda-http` |
| Overlay directory | `<manifests_path>/previews/<app>/` |
| Image tags | `ghcr.io/<caller owner>/<image>:pr-<PR>`, digest-pinned into the overlay |

`k8s-preview` inputs: `app`, `images` (JSON array of
`{name, dockerfile, var, build_args}`, where `var` is the `${...}` name the overlay expects
for that image's digest), `deployments` (JSON array), `preview_url` (template with
`${PR_NUMBER}`/`${LOCAL_DOMAIN}`), `manifests_repo`, plus optional `manifests_path`,
`overlay_path`, `copy_secrets`, `referencegrant`.

The overlay is rendered with `kubectl kustomize --load-restrictor LoadRestrictionsNone` and
envsubst'ed with `${PR_NUMBER}`, `${LOCAL_DOMAIN}` and one `<var>` per image. Secrets are never
rendered by the overlay — `copy_secrets` copies them from the app's namespace. The
ReferenceGrant is applied separately (it must stay in `keda-http`, but the overlay's global
`namespace:` transformer would relocate it) and deleted by name on teardown, so it is not one
of the overlay kustomization's `resources`.

## Pinning

Callers reference `@main`. There is no release or tag step: the workflows are consumed as-is, so a merge here lands in every caller on its next run.

To raise the tool versions, bump `KUSTOMIZE_VERSION` / `KUBECONFORM_VERSION` / `KUBERNETES_VERSION` in the workflow `env:` block — every caller picks it up with no change on their side.
