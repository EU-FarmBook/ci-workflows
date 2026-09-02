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

**The workflow validates before committing.** It runs `kustomize build
overlays/dev` and rejects empty image tags and unresolved placeholders. An
invalid manifest blocks Argo CD from rendering the whole overlay, which stops
deployment for every service in that application, not only the one that changed.

**Concurrency.** Runs are serialised per service. Pushes to the platform
repository are rebased and retried to tolerate concurrent deployments of
different services.
