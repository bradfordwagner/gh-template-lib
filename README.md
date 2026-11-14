# gh-template-lib

Reusable GitHub Actions workflows that build multi-arch container images from a `config.yaml`, for the
`gh-template-*` repos and the images made from them. It replaces the Dagger module
[`dagger-container-builds`](https://github.com/bradfordwagner/dagger-container-builds) with plain `docker buildx`.

Each flavor is one reusable workflow, named after the `flavor:` in `config.yaml`:

| Flavor | Workflow | Status |
|---|---|---|
| `mirror` | `.github/workflows/mirror.yml` | implemented |
| `custom` | `.github/workflows/custom.yml` | implemented |
| `ansible_playbook` | `.github/workflows/ansible_playbook.yml` | implemented |
| `ansible_role` | `.github/workflows/ansible_role.yml` | implemented (test only, pushes nothing) |

Every flavor runs the same pipeline: `product` expands `builds × archs` into a matrix, one `builds` job per entry
pushes `<target_image>-<os>_<arch>`, and `manifest` combines those into the multi-arch `<target_image>`.

## Usage

`container_branches.yml` in a mirror template:

```yaml
name: container-branches
on:
  push:
    branches:
      - '**'
    tags-ignore:
      - '**'
  workflow_dispatch: {}
jobs:
  container:
    uses: bradfordwagner/gh-template-lib/.github/workflows/mirror.yml@0.1.0
    permissions:
      contents: read
      packages: write
    with:
      version: latest
```

`container_tags.yml` is the same with `on: push: tags: ['*']` and `version: ${{ github.ref_name }}`.

The caller job must grant `packages: write`: a called workflow can only narrow the caller's token, never widen
it. Images are pushed to ghcr.io with the caller's `GITHUB_TOKEN`; no secrets are needed.

## `config.yaml` (mirror)

```yaml
target_repo: ghcr.io/bradfordwagner/template-mirror
flavor: mirror
builds:
  - repo: alpine
    repo_override: ""        # optional: replaces repo in the target tag
    tag: "3.20"
    archs:
      - linux/amd64
      - linux/arm64
    commands:                # optional: one RUN line each
      - apk add --no-cache bash
```

Pushes `ghcr.io/bradfordwagner/template-mirror:<version>-alpine_3.20` (multi-arch) and the per-arch tags
`...-alpine_3.20-linux_amd64` and `...-alpine_3.20-linux_arm64`. When `commands` is set and the repo has a
non-empty `scripts/` directory, it is copied to `/src/scripts`, made executable, and removed after the commands.

## `config.yaml` (ansible_playbook)

```yaml
target_repo: ghcr.io/bradfordwagner/gh-template-ansible-playbook
flavor: ansible_playbook
builds:
  - os: alpine_3.24
    archs:
      - linux/amd64
      - linux/arm64
upstream:
  repo: ghcr.io/bradfordwagner/ansible
  tag: 6.6.0
```

Each entry builds `FROM <upstream.repo>:<upstream.tag>-<os>`, copies the repo (minus `.git`) into `/src`, then runs
`ansible-galaxy install -r` for `requirements.yml` and `meta/requirements.yml`, and `ansible-playbook` for `test.yml`
and `playbook.yml`, each only if present, in that order. Like Dagger, it sets no `WORKDIR`: the commands run in
the upstream image's own (`/src` for `ghcr.io/bradfordwagner/ansible`). Pushes
`<target_repo>:<version>-<os>` (multi-arch) and its per-arch tags. Inputs are the same as mirror's.

## Inputs (mirror)

| Input | Default | |
|---|---|---|
| `version` | `latest` | Tag prefix. Branch builds pass `latest`, tag builds the ref name. |
| `config` | `config.yaml` | Path to the config in the caller's repo. |
| `arc` | `auto` | `auto`: arm64 entries run on `arc_runner_label` while the caller repo's `ARC_HOSTS` variable is non-empty. `on` / `off` force it. |
| `arc_runner_label` | `arc-arm64` | ARC scale set label. |
| `buildx_driver` | `docker` | `docker` builds on the runner's daemon. `remote` is the seam for a shared BuildKit. |
| `buildx_endpoint` | `''` | BuildKit address for `remote`. |
| `registry_cache` | `false` | On ARC jobs only, pull docker.io and ghcr.io bases through the registry pull-through caches. |
| `registry_cache_dockerio` | `host.k3d.internal:5001` | docker.io cache. |
| `registry_cache_ghcr` | `host.k3d.internal:5002` | ghcr.io cache. |
| `buildkit_version` | pinned | `moby/buildkit` tag, used only by the `docker-container` driver. |

Outputs: `product` (the matrix JSON) and `images` (the multi-arch tags pushed).

## `config.yaml` (custom)

```yaml
target_repo: ghcr.io/bradfordwagner/gh-template-custom-image
flavor: custom
upstream:
  repo: ghcr.io/bradfordwagner/base
  tag: 4.7.1
builds:
  - os: alpine_3.24
    archs:
      - linux/amd64
      - linux/arm64
    args:                    # optional: each key becomes a --build-arg
      pkg_installer: alpine
```

Builds the caller's own `Dockerfile` with the repo root as context, passing every `args` key plus
`OS=<upstream.repo>:<upstream.tag>-<os>` (here `ghcr.io/bradfordwagner/base:4.7.1-alpine_3.24`) as build args.
When two builds share an `os`, both use the first one's `args`. Pushes
`ghcr.io/bradfordwagner/gh-template-custom-image:<version>-alpine_3.24` (multi-arch) and its per-arch tags.
Inputs are mirror's. Upstream bases are on ghcr.io, so on ARC only `registry_cache: true` sends them through a cache.

### ARC routing

The `product` job reads `vars.ARC_HOSTS`. In a reusable workflow that is the **caller** repo's variable
(GitHub: "For reusable workflows, the variables from the caller workflow's repository are used"), so the toggle
that `repos:on` / `repos:off` flips on each template keeps working with no input. arm64 entries get
`runner: arc-arm64`; everything else keeps the hosted runner. The template's own `runs-on` override goes away.

On ARC the job still needs Docker: keep `dind: true` for the repo until a shared BuildKit replaces it.

### Registry caches

Off by default. With `registry_cache: true`, jobs on `arc_runner_label` switch the `docker` driver to
`docker-container` (BuildKit in a container on the dind daemon, host network) with `buildkitd.toml` mirrors for
docker.io and ghcr.io pointing at the two plain-HTTP caches. Hosted jobs ignore it, since they cannot reach
`host.k3d.internal`. The dind daemon already mirrors docker.io on its own, so the `docker` driver gets docker.io
caching without this input; ghcr.io bases need it.

## `ansible_role`

Test only, as under Dagger: no image is pushed and there is no `manifest` job, so the caller grants only
`contents: read`. Each `builds` job runs a throwaway `docker buildx build` (no output) of:

```dockerfile
FROM <upstream.repo>:<upstream.tag>-<os>
COPY . /src
RUN ["ansible-galaxy","install","-r","requirements.yml"]       # each line only if the file exists
RUN ["ansible-galaxy","install","-r","meta/requirements.yml"]
RUN ["ansible-playbook","test.yml"]
RUN ["ansible-playbook","playbook.yml"]
```

The commands run in the upstream image's own workdir (`/src` for `ghcr.io/bradfordwagner/ansible`), as Dagger
ran them. The `product` entry fields, in order: `arch`, `os`, `runner`, `target_image`, `upstream_image`,
`index`, `display` (`<index>:<os>-<version>(<arch>)`). Inputs are mirror's, minus the ghcr.io login; outputs:
`product` only. Galaxy import stays a job in the caller, `needs: container`.

## Development

```sh
./test/mirror-product.sh   # needs yq v4 and jq
./test/ansible_playbook-product.sh
./test/custom-product.sh
./test/ansible_role-product.sh
yamllint --strict .
actionlint
```

See `CLAUDE.md` for how the repo is laid out and `docs/migration.md` for moving templates off Dagger.
