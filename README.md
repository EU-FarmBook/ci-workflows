# ci-workflows

Shared GitHub Actions workflows for EU-FarmBook services.

Each service repository calls `build-and-deploy.yml` rather than carrying its
own build and deployment logic.

## build-and-deploy.yml

On a push to `main`, the workflow builds the service image, tags it with the
commit SHA, pushes it to GHCR, and records that tag in the `dev` branch of
`eufarmbook-platform`. Argo CD deploys the change to DEV.

Production is not affected. Promotion requires a `dev` → `main` merge in
`eufarmbook-platform` followed by a manual sync in Argo CD.

### Usage

`.github/workflows/deploy.yml` in the service repository:

```yaml
name: deploy

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  deploy:
    permissions:
      contents: read
      packages: write
    uses: EU-FarmBook/ci-workflows/.github/workflows/build-and-deploy.yml@main
    with:
      service: pagesense
    secrets: inherit
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `service` | yes | | Directory under `base/apps/` in `eufarmbook-platform`, and the image name in GHCR |
| `dockerfile` | no | `Dockerfile` | Path to the Dockerfile |
| `context` | no | `.` | Build context |
| `build_args` | no | | Newline-separated build arguments |
| `platform_ref` | no | `dev` | Branch of `eufarmbook-platform` to write the tag to |

Examples:

```yaml
    with:
      service: agri-gate
      dockerfile: deploy/docker/Dockerfile

    with:
      service: agri-tag
      build_args: |
        PRELOAD_AGRI_MODEL=true
```

## Requirements

**`permissions` on the calling job.** Default workflow token permissions are
read-only. A called workflow cannot be granted more than its caller holds, so
omitting the block causes the run to fail at startup with no job and no log.

**`PLATFORM_REPO_TOKEN`.** An organisation secret: a fine-grained token with
`Contents: write` on `eufarmbook-platform` only, scoped to the repositories that
deploy. Registry credentials are not stored; `GITHUB_TOKEN` is minted per job.

The token expires. When it does, deployments fail at the platform checkout step
with no other warning.

**GHCR package access.** The package must grant `Write` to its repository under
*Package settings → Manage Actions access*. Packages first pushed from a
workstation are not linked to a repository and will reject `GITHUB_TOKEN` with
`denied: permission_denied`.

Adding `LABEL org.opencontainers.image.source="https://github.com/EU-FarmBook/<repo>"`
to the Dockerfile links the package automatically, avoiding the manual step for
services whose first push comes from CI.

## Notes

**Image tags are commit SHAs, never `latest`.** Argo CD detects change by
diffing manifests rather than polling the registry, so a moving tag would not
trigger a deployment, and a restarted pod could silently run different code.

**The workflow validates before committing.** Each environment is one
kustomization per Argo CD Application, so it builds every `overlays/<env>/*`
directory (skipping `_env` and `_shared`, which are not Applications) and
rejects empty image tags and unresolved placeholders. It pins the same kustomize
version Argo CD's repo-server runs: validating with a different one can pass
something Argo cannot render, which is the failure this step exists to prevent.

An invalid manifest stops Argo CD rendering that Application. Since the 6 Sep
split that is one service rather than the whole namespace — but it is still the
difference between a failed deploy and a deployed service.

**Concurrency.** Runs are serialised per service. Pushes to the platform
repository are rebased and retried to tolerate concurrent deployments of
different services.

## Maintenance

**Versions (2 Oct 2026).** `actions/checkout@v7`, `docker/setup-buildx-action@v4`,
`docker/login-action@v4`, `docker/build-push-action@v7`: the current majors, all on
Node 24. The runner is pinned to `ubuntu-24.04` rather than `ubuntu-latest`, which
GitHub moves to Ubuntu 26 from 19 October 2026. Move it deliberately: change the
label on `main`, run one `@main` caller by hand (`gh workflow run deploy.yml
--repo EU-FarmBook/<repo>`), and only then move `v1`.

**Who uses what.** Callers pin either `@v1` (most) or `@main` (agri-tag, agri-gate,
omnilingua at the time of writing). A push to `main` reaches the `@main` callers at
their next run; `v1` is a tag and reaches the rest only when it is moved:

```bash
git tag -f v1 && git push -f origin v1
```

Test on an `@main` caller first, then move the tag, then run one `@v1` caller.
A rerun of an unchanged commit rebuilds from cache, finds the tag already
recorded, and commits nothing, so it is a safe test.

**The render check must be able to fail.** It renders every environment named
in its loop and fails when one is missing or renders nothing. Until 2 Oct 2026 it
named `qlt`, renamed to `uat` on 16 September, and passed for an environment it
never looked at. When an environment is added or renamed in the platform
repository, change the loop here in the same week.

