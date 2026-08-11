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

## Building locally

```
crossplane xpkg build --package-root=. --examples-root=examples --package-file=crossplane-compositions.xpkg --ignore=".github/workflows/publish.yaml,deploy/configuration.yaml"
```

CI pushes every `main` commit to `ghcr.io/entr0pian/crossplane-compositions:<sha>` and to
the floating `:main` tag. `deploy/configuration.yaml` — the `Configuration` CR applied to
whichever cluster runs this — tracks `:main` for now (`packagePullPolicy: Always`, so new
pushes get picked up on the next reconcile). No version pinning yet; that's a deliberate
"get it working first" simplification, same as the `argocd-write-token` credential reuse
was before it got its own dedicated secret — a SHA-pinned, Helm-parameterized version
(matching how `operator`/`applicationRepositoryOperator` do it) is the follow-up once this
composition is more than a proof of concept.

GHCR packages default to private on first push — flip the package's visibility to
public in GitHub package settings after the first successful run, since Argo CD doesn't
need a `packagePullSecret` for this.

## Not yet wired up

- No Argo CD Application yet — `taskapp-argocd` doesn't reference this repo, so
  `deploy/configuration.yaml` isn't applied anywhere through GitOps yet. It can be
  applied directly (`kubectl apply -f deploy/configuration.yaml` against management) as
  a manual smoke test in the meantime, once the package is pushed and public.
