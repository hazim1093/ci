# ci

Shared, reusable GitHub Actions workflows for hazim1093's repositories. Public by design — nothing repo-specific or private lives here; callers pass their own paths via `inputs`.

Version pins (kustomize, kubeconform, kubernetes) live here, so tool bumps happen in one place.

Each workflow is standalone: independent triggers, no `needs:` coupling, usable alone in any repo — k8s repos, app repos, docs repos. When a repo calls both, they run as two parallel checks.

## Reusable workflows

### k8s-validation

Kustomize-builds and kubeconform-validates every changed manifest unit directory under the given roots (Flux postBuild placeholders are substituted with sample values first).

Callers pass the manifest roots they manage, space-separated:

```yaml
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

## Versioning

Releases are automated from Conventional Commits (release-please):

- `fix` → patch, `feat` → minor, `feat!`/`BREAKING CHANGE:` → major
- On merge to `main`, release-please opens a `chore(main): release vX.Y.Z` PR with the changelog; merging it cuts the tag + GitHub Release and moves the floating major tag (`v1`)
- PR titles are validated against Conventional Commits, since squash-merge titles become the release commits

Callers pin the floating major — `@v1` — so fixes and features flow automatically while breaking changes land as `@v2` and are adopted deliberately.
