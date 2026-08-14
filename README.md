# crossplane-compositions

Source-of-truth repo for Crossplane `Configuration` packages, built and published as OCI
artifacts to GHCR instead of applied as raw YAML through Helm/Argo CD:

```
GitHub repo → GitHub Actions → GHCR → Argo CD (Configuration CR) → Crossplane
```

Argo CD's role shrinks to applying one small `Configuration` CR per package; Crossplane's
own package manager pulls, versions, and resolves the `dependsOn` providers/functions
declared in `crossplane.yaml`.

This repo is meant to grow to hold all compositions over time. It starts with a single
one below; the existing RDS/SQS compositions (`helm-charts/crossplane-compositions`,
built on Crossplane 1.x cluster-scoped resources) stay where they are — disabled for now
— until they're reworked for the namespaced-XR model and moved in here.

## Packages

### `apis/githubrepository` — `GitHubRepository`

Namespaced XR (`repo.taskapp.io/v1alpha1 GitHubRepository`) that composes a
`repo.github.m.upbound.io/v1alpha1 Repository` via `crossplane-contrib/provider-upjet-github`.

Provider credentials are wired up separately, outside this repo: `helm-charts/crossplane-provider-config`
applies the `github.upbound.io` `ProviderConfig` (gated by its own `github.enabled` toggle),
referencing a dedicated `crossplane-github-credentials` Secret delivered via ESO from
`taskapp/platform/crossplane-github-token` in AWS Secrets Manager (see
`helm-charts/platform`'s `crossplaneGithub.secretPath`). This package only depends on the
provider being installed — it doesn't carry or apply any credentials itself.

## Delivery model

This repo only builds and releases the OCI `Configuration` package — it does not decide
where or when that package gets installed. That split is deliberate:

```
crossplane-compositions   → WHAT CAN BE RELEASED (this repo)
application-repositories  → WHAT VERSION SHOULD RUN WHERE
argocd                    → HOW THAT DESIRED STATE IS DELIVERED
```

Publishing a new version here never deploys it anywhere by itself. A cluster only picks
up a new version when `application-repositories`' `packages/crossplane-compositions/<env>.yaml`
is updated to reference it — see that repo's README for the package contract shape, and
`argocd`'s `taskapp-packages` ApplicationSet for how it's installed.

### `charts/configuration-installer/`

A tiny installer-adapter Helm chart, versioned and released independently of the
`Configuration` package itself. It converts generic values (`name`, `package.repository`,
`package.version`, `pullPolicy`) into a single `pkg.crossplane.io/v1 Configuration`
resource — nothing else. It intentionally contains no XRDs or Compositions; those only
ever ship inside the OCI package built from this repo's `apis/`.

## Releasing

Releases are triggered by pushing a Git tag matching `v*` — the tag becomes the OCI tag
directly:

```
git tag v0.1.0
git push origin v0.1.0
```

`.github/workflows/release.yaml` then builds the package and pushes
`ghcr.io/entr0pian/crossplane-compositions:v0.1.0` (plus a floating `:latest` alias, for
convenience only — `application-repositories` always references the immutable version,
never `:latest`). `.github/workflows/validate.yaml` runs on every PR that touches
`crossplane.yaml`, `apis/**`, or the installer chart — it builds the package and lints
the chart, but never logs in to GHCR or pushes anything.

## Building locally

```
crossplane xpkg build --package-root=. --examples-root=examples --package-file=crossplane-compositions.xpkg --ignore="charts/configuration-installer/Chart.yaml,charts/configuration-installer/values.yaml,charts/configuration-installer/templates/configuration.yaml,.github/workflows/validate.yaml,.github/workflows/release.yaml"
```

(The `crossplane` CLI's `--ignore` flag doesn't support recursive globs like
`charts/**` — list exact file paths instead.)

GHCR packages default to private on first push — flip the package's visibility to
public in GitHub package settings after the first successful release, since Argo CD
doesn't need a `packagePullSecret` for this.
