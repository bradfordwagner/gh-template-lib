# Migrating from `dagger-container-builds`

## Why

On ARC every job started its own Dagger engine inside its dind sidecar. Two failures followed from that:

1. **Disk exhaustion.** Each runner pod pulled and unpacked its own engine image and kept its own engine state,
   so concurrent builds filled the node's disk.
2. **An engine start-up race.** The engine's DNS (dnsmasq behind the CNI `dnsname` plugin, `10.87.0.1`) is not
   up yet when the module's first git exec resolves `github.com`. Under load that fails instantly with
   `Could not resolve host: github.com`, several jobs in the same second.

`docker buildx` with the `docker` driver builds on the daemon the job already has. No engine image, no second
DNS server, no module fetched from GitHub at job start.

## Decisions

These are working defaults. Each is marked decided where it applies below; none is decided.

| Decision | Chosen | Alternative |
|---|---|---|
| Galaxy publish for `ansible_role` | Stays a job in the template, `needs: container`; the lib never holds `GALAXY_TOKEN` | `galaxy_publish` input in the lib, caller passes the secret |
| Lib versioning | Semver tags from `main`, first tag `0.1.0`; breaking input or product-contract change = major bump | — |
| Dagger quirks | Kept unchanged for parity: only exactly `linux/arm64` gets an arm runner (`linux/arm64/v8` falls to `ubuntu-26.04`); `display` uses `repo`, not `repo_override`; builds sharing a `target_image` get one combined manifest | Fix them in a later major |
| buildx driver | `docker` (runner's own daemon: no BuildKit image pull, least disk, inherits dind's docker.io mirror) | `docker-container` |
| Manifest media types | OCI everywhere, as Dagger pushed: `oci-mediatypes=true`, and hosted runners use `docker-container` (their classic image store would push Docker v2). ARC's dind already pushes OCI with `docker` | Docker v2 on hosted |
| Template tags | Finalize the in-flight Dagger ARC betas first, then the lib switch is the next minor's betas (below) | Continue the in-flight version as `-beta2` |

## Rollout scope

1. **The four templates only:** `gh-template-mirror-image`, `gh-template-custom-image`,
   `gh-template-ansible-playbook`, `gh-template-ansible-role`. Nothing downstream moves yet.
2. **ARC is in scope from the start.** arm64 entries run on `arc-arm64` while the template's `ARC_HOSTS`
   variable is non-empty, else on hosted `ubuntu-26.04-arm` (the lib's default `arc: auto`). amd64 stays hosted.
3. **One OS while testing.** Each template's `config.yaml` is cut to a single OS that builds both
   `linux/amd64` and `linux/arm64`, so a run is one amd64 job, one arm64 job and one manifest. Once that is green
   on a beta tag, restore the full OS list in a follow-up commit and cut another beta.
4. **Beta tags first.** Each template releases `X.Y.Z-betaN` tags until the full OS list is green, then the
   final `X.Y.Z`.

   Current state: the Dagger ARC routing is merged on each `main` and beta-tagged, all tag builds green:
   mirror `2.4.0-beta1`, custom `0.3.0-beta1`, ansible_playbook `2.6.0-beta1`, ansible_role `0.6.0-beta1`.
   **Done:** cut as final (`2.4.0`, `0.3.0`, `2.6.0`, `0.6.0`) so the last Dagger
   release is a clean rollback point, then release the lib switch as the next minor's betas: mirror
   `2.5.0-betaN`, custom `0.4.0-betaN`, ansible_playbook `2.7.0-betaN`, ansible_role `0.7.0-betaN`. Tag builds pass the ref name as `version`, so a beta pushes `<target_repo>:X.Y.Z-betaN-<os>`
   and never touches `latest-*` or a released tag.
5. **Downstream follows the templates.** `container-ansible`, the `ansible-role-*` repos and every other repo made
   from a template switch only after that template's beta tags pass with the full OS list.

### Flavor order

1. `mirror`: simplest; one Dockerfile generated from config, no consumer Dockerfile. Implemented.
2. `custom` and `ansible_playbook`: build args and an upstream image on ghcr.io; ansible needs the
   `ansible-galaxy` / `ansible-playbook` sequence reproduced as Dockerfile `RUN` steps.
3. `ansible_role` last: its tag runs import the role into Galaxy, and every `ansible-role-*` repo inherits the
   template, so a mistake there is public and multiplied.

Per flavor: implement it here with fixtures generated from the Dagger module's Go code, tag the lib, then on the
template: one-OS config plus the new workflows on a branch, green branch run, beta tag, full OS list, beta tag,
final tag.

## Template diffs

Base: each template's `main` after the Dagger ARC-routing merges (mirror `18face3`, custom `c577fbe`,
ansible_playbook `3a08f99`, ansible_role `a35835c`). Each template changes three files.

### Workflows (all four templates)

Replace everything under `jobs:` in both workflows with one `container` job, and drop the workflow-level `env:`
(`DAGGER_VERSION`, `version`). The Dagger install, the `runs-on` ARC override, the engine override and the
ghcr.io login all go away; the lib does them. Per template that is about -98 / +7 lines per file
(ansible_role: -62 / +6 and -63 / +7).

`.github/workflows/container_branches.yml` after, with `<flavor>` = `mirror`, `custom`, `ansible_playbook` or
`ansible_role`:

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
    uses: bradfordwagner/gh-template-lib/.github/workflows/<flavor>.yml@0.1.0
    permissions:
      contents: read
      packages: write
    with:
      version: latest
```

`.github/workflows/container_tags.yml` after:

```yaml
name: container-tags
on:
  push:
    tags:
      - '*'
jobs:
  container:
    uses: bradfordwagner/gh-template-lib/.github/workflows/<flavor>.yml@0.1.0
    permissions:
      contents: read
      packages: write
    with:
      version: ${{ github.ref_name }}
```

`gh-template-ansible-role` differs: it pushes nothing, so both files drop `packages: write`, and
`container_tags.yml` keeps its `ansible-role-publish` job unchanged except `needs: builds` becomes
`needs: container`. Galaxy publish stays in the template (decided).

The lib pin `@0.1.0` assumes the versioning default (decided); the driver is the lib's
`buildx_driver: docker` default, so templates pass no driver input (decided).

All four flavors are implemented.

### `config.yaml`: one OS while testing

Keep exactly one `builds` entry, with both arches. `target_repo`, `flavor` and `upstream` are unchanged.

| Template | Keep | Product |
|---|---|---|
| `gh-template-mirror-image` | `repo: alpine`, `tag: "3.20"` | 2 entries; fixture `gh-template-mirror-image-beta` (`latest`, `2.5.0-beta1`, ARC on/off) |
| `gh-template-custom-image` | `os: alpine_3.24` (`pkg_installer: alpine`) | 2 entries |
| `gh-template-ansible-playbook` | `os: alpine_3.24` | 2 entries |
| `gh-template-ansible-role` | `os: alpine_3.24` | 2 entries |

`gh-template-mirror-image/config.yaml` after:

```yaml
target_repo: ghcr.io/bradfordwagner/template-mirror
flavor: mirror
builds:
    - repo: alpine
      repo_override: ""
      tag: "3.20"
      archs:
        - linux/amd64
        - linux/arm64
```

The others keep the same shape: the one `- os: alpine_3.24` item and its `archs` (and `args` for custom).
Restoring the full list is `git revert` of that commit, or `git checkout <base> -- config.yaml`.

Note: `gh-template-ansible-role`'s `target_repo` says `gh-template-ansible-playbook`. It is unused because
ansible_role pushes nothing, but fix it before anything ever publishes from that flavor.

### Other files

- `renovate.json`: unchanged. Renovate's github-actions manager tracks the `uses:` ref and, with
  `pinGitHubActionDigests`, pins it to a digest with the tag in a comment. Expect the mirror/custom presets to
  open PRs re-adding OS versions while the config is cut down; close or hold them until the list is restored.
- Check names change from `product`, `<display>`, `manifest-tool` to `container / product`,
  `container / <display>`, `container / manifest`. Update branch protection if any of them is required.
- Repo list in k8s-deployments: keep `dind: true`. The `docker` driver needs a daemon on ARC.
- `workflow.yaml` (Argo), `Dockerfile`, `install.sh`, playbooks and role files are unchanged.

### Flavor notes

- **custom:** the lib passes every `builds[].args` key plus `OS=<upstream.repo>:<upstream.tag>-<os>` as build
  args, as Dagger did. Implemented; the template's `manifest-tool` job (its own Dagger install) is replaced by the lib's
  `manifest`. Upstream bases are on ghcr.io; set `registry_cache: true` if ARC pulls should go through
  the cache.
- **ansible_playbook:** Dagger copied the repo onto `/src` and ran `ansible-galaxy` / `ansible-playbook` by
  relative path without setting a workdir, so they ran in the upstream image's `WORKDIR`. For
  `ghcr.io/bradfordwagner/ansible` that is `/src` (its Dockerfile: `COPY . /src`, `WORKDIR /src`), so the paths
  resolve to the consumer's files, copied over what the upstream left in `/src`. The lib's generated Dockerfile
  does the same: `COPY . /src`, no `WORKDIR`, exec-form `RUN`. It excludes `.git` from the copy.
- **ansible_role, Galaxy (decided: template job):** keeping the publish job in the template
  means the lib never holds `GALAXY_TOKEN`. The alternative is a `galaxy_publish: true` input with the caller passing
  `secrets: GALAXY_TOKEN: ${{ secrets.GALAXY_TOKEN }}`: one place to change the Galaxy action, but every role
  repo's release then depends on a lib tag. A beta tag on the role template imports a beta into Galaxy, so
  decide whether beta tags should skip the publish job (e.g. `if: ${{ !contains(github.ref_name, '-beta') }}`).
  Decided: beta and final tags both publish; the job stays as is.

## Downstream, after the templates

`container-ansible` (custom; it publishes `ghcr.io/bradfordwagner/ansible`, the upstream of both ansible
flavors) and the repos made from each template switch with the same workflow diff, full OS list, once that
template's beta tags have passed with its full OS list. They follow the same beta-then-final tagging.

## Running old and new side by side

Beta tags already keep the lib's tag builds apart from released images. Branch builds are the risk: both
push `latest-*`, and the last writer wins, so never run Dagger and the lib on the same trigger. To compare against
Dagger before a template switches, preview the lib under a different `version` from a dispatch-only workflow on
the template's branch:

```yaml
# .github/workflows/container_lib_preview.yml
name: container-lib-preview
on:
  workflow_dispatch: {}
jobs:
  container:
    uses: bradfordwagner/gh-template-lib/.github/workflows/mirror.yml@main
    permissions:
      contents: read
      packages: write
    with:
      version: lib-preview
```

That pushes `<target_repo>:lib-preview-<repo>_<tag>` and its per-arch tags next to Dagger's `latest-...`.
Compare:

```sh
for t in alpine_3.20 ubuntu_noble; do
  docker buildx imagetools inspect ghcr.io/bradfordwagner/template-mirror:latest-$t
  docker buildx imagetools inspect ghcr.io/bradfordwagner/template-mirror:lib-preview-$t
done
```

Expect the same platforms and the same base layers. Media types match: OCI, as Dagger pushed. Delete the `lib-preview-*` versions from the
package afterwards and remove the preview workflow before merging.

The matrix itself is already checked offline: `test/mirror-product.sh` compares the lib's `product` to the
Dagger module's output for this template's `config.yaml`.

## Rollback

Revert the template commit. The Dagger module stays at `0.4.1`, untouched, and the reverted workflows pin it
again. With the tag default above, the final Dagger ARC tags (`2.4.0`, `0.3.0`, `2.6.0`, `0.6.0`) are the
last-known-good releases to point consumers back to. Images keep their tags either way, so a rollback rebuilds over them with Dagger on the next push. Lib tags
are never moved; fix forward with a new tag and bump the pin.

Keep `dagger-container-builds` and the Renovate Dagger managers until the last consumer has switched and run a
tag build green; archive the module repo after that.

## `infra/docs/arc-runners.md` afterwards

- *Image builds: `dagger-container-builds` today*: becomes "Image builds: `gh-template-lib`". Remove the
  per-consumer `runs-on` override, `DAGGER_VERSION` and `_EXPERIMENTAL_DAGGER_RUNNER_HOST` steps. Builds pull
  through dind's own docker.io mirror; ghcr.io pulls go through the cache only with `registry_cache: true`.
  "Build cache: none survives" still holds.
- *Routing*: note that templates no longer carry the override; the lib's `product` job reads the caller's
  `ARC_HOSTS`.
- *Rollout to every consumer, and an engine DNS flake* (gha-7at) and the DiskPressure eviction (gha-7g8):
  close them once a concurrent run of all consumers on the lib is green, with that run's ID as evidence. Until
  then, "fixed by removing the engine" is a hypothesis.
- *From `dagger-container-builds` to an actions lib*: mark done per flavor; name the repo `gh-template-lib`.
- *Day 2: Shared BuildKit*: the lib's seam is `buildx_driver: remote` plus `buildx_endpoint`. mTLS client
  certificates are not wired yet; add inputs/secrets for them when buildkitd exists. Removing `dind: true`
  then becomes possible per repo.
- *Docker-in-Docker is a known gap*: unchanged until the shared BuildKit lands.
