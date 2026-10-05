# ci-workflows

Shared GitHub Actions workflows for EU-FarmBook services.

Each service repository calls `build-and-deploy.yml` rather than carrying its
own build and deployment logic.

## build-and-deploy.yml

The workflow builds the service image, tags it with the commit SHA, pushes it
to GHCR, and records that tag in `eufarmbook-platform`. Where it records the tag
depends on `branch_model`.

| `branch_model` | Branch pushed | Tag recorded in | Deploys to |
|---|---|---|---|
| `legacy` (default) | whatever the caller triggers on | platform `dev`, `base/apps/<service>/deployment.yaml` | DEV |
| `dev-main` | a branch in `dev_branches` (default `dev`) | platform `dev`, `base/apps/<service>/deployment.yaml` | DEV |
| `dev-main` | `main` | platform `main`, `overlays/uat/_images` and `overlays/prd/_images` | UAT at once, PRD after a manual Sync |
| `dev-main` | anything else | nothing: the run fails before building | - |

With `dev-main`, every service repository has a `dev` branch for DEV and a
`main` branch for UAT and PRD. `legacy` is the earlier behaviour, kept as the
default so that a repository changes only when its own `deploy.yml` opts in.

A release from `main` reuses the image DEV already built when `main` points at a
commit DEV built, so UAT and PRD run the bytes DEV tested. A merge commit is a
new commit and is built.

PRD is never changed by a workflow run. Its Argo CD Applications are not
automated, so a recorded release waits for someone to press Sync.

### Usage

`.github/workflows/deploy.yml` in the service repository:

```yaml
name: deploy

on:
  push:
    branches: [ dev, main ]
  workflow_dispatch:

jobs:
  deploy:
    permissions:
      contents: read
      packages: write
    uses: EU-FarmBook/ci-workflows/.github/workflows/build-and-deploy.yml@v1
    with:
      service: pagesense
      branch_model: dev-main
    secrets:
      PLATFORM_REPO_TOKEN: ${{ secrets.PLATFORM_REPO_TOKEN }}
```

A repository that still calls its development branch something else lists both
names until it is renamed, in the trigger and in `dev_branches`:

```yaml
on:
  push:
    branches: [ dev, develop, main ]
...
    with:
      branch_model: dev-main
      dev_branches: dev develop
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `service` | yes | | Directory under `base/apps/` in `eufarmbook-platform`, and the image name in GHCR |
| `dockerfile` | no | `Dockerfile` | Path to the Dockerfile |
| `context` | no | `.` | Build context |
| `build_args` | no | | Newline-separated build arguments |
| `branch_model` | no | `legacy` | `legacy` or `dev-main`, as above |
| `dev_branches` | no | `dev` | `dev-main` only: space-separated branches that deploy to DEV |
| `platform_ref` | no | `dev` | `legacy` only: branch of `eufarmbook-platform` to write the tag to |

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
*Package settings > Manage Actions access*. Packages first pushed from a
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
directory (skipping `_env`, `_shared` and `_images`, which are not Applications) and
rejects empty image tags and unresolved placeholders. It pins the same kustomize
version Argo CD's repo-server runs: validating with a different one can pass
something Argo cannot render, which is the failure this step exists to prevent.

An invalid manifest stops Argo CD rendering that Application, so the service
is not deployed.

**Concurrency.** Runs are serialised per service. Pushes to the platform
repository are rebased and retried to tolerate concurrent deployments of
different services.

## Maintenance

**Versions.** `actions/checkout@v7`, `docker/setup-buildx-action@v4`,
`docker/login-action@v4`, `docker/build-push-action@v7`: the current majors, all on
Node 24. The runner is pinned to `ubuntu-24.04` rather than `ubuntu-latest`. To move it:
change the label on `main`, run one `@main` caller by hand (`gh workflow run
deploy.yml --repo EU-FarmBook/<repo>`), and only then move `v1`.

**Who uses what.** Callers pin either `@v1` (most) or `@main` (agri-gate, agri-tag,
euf_metadata_translations, omnilingua, project_pages_translations). A push to `main` reaches the `@main` callers at
their next run; `v1` is a tag and reaches the rest only when it is moved:

```bash
git tag -f v1 && git push -f origin v1
```

Test on an `@main` caller first, then move the tag, then run one `@v1` caller.
A rerun of an unchanged commit rebuilds from cache, finds the tag already
recorded, and commits nothing, so it is a safe test.

**The release files are CI's.** CI rewrites everything from the `images:` line
to the end of `overlays/{uat,prd}/_images/kustomization.yaml`. An empty list is
written `images: []`, because kustomize rejects a Component with no fields. After
recording, the render check confirms that every reference to the image in the
target environments carries the new tag. A release written where no Application
reads it would otherwise pass and change nothing.

**Keep the render loop in step with the platform.** The render check renders
every environment named in its loop and fails when one is missing or renders
nothing. When an environment is added or renamed in the platform repository,
update the loop here.

