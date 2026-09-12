# go-module-update-workflow

A reusable GitHub Actions workflow for Nix flakes that package tagged Go
versions with `buildGoModule`. Upstream versions can be discovered from either
Git tags or published GitHub Releases.

For every selected upstream version that is not already present as a tag in the
caller repository, the workflow:

1. updates `tag` and `commit` in the caller's flake;
2. asks Nix for the new `vendorHash`;
3. verifies the build;
4. creates a commit and matching Git tag; and
5. atomically pushes the commit and tag to the caller's default branch.

Each successfully packaged version is pushed independently, in version order.

## Caller

Keep triggers, permissions, and concurrency in the caller repository:

```yaml
name: Update example

on:
  schedule:
    - cron: "0 6 * * *"
      timezone: "Asia/Tokyo"
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: update-example
  cancel-in-progress: false

jobs:
  update:
    uses: ixpkgs/go-module-update-workflow/.github/workflows/update-go-module.yml@COMMIT_SHA
    with:
      upstream_repository: https://github.com/example/example.git
      package_name: Example
```

Pin the reusable workflow to a full commit SHA. The caller's `GITHUB_TOKEN` is
used to push updates; no additional secret is required.

Optional inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `discovery_mode` | `tags` | Version source: `tags` or `releases` |
| `upstream_github_repository` | empty | GitHub `owner/repository`; required in `releases` mode |
| `include_prereleases` | `false` | Include GitHub prereleases in `releases` mode |
| `tag_pattern` | semantic `v` tags | Bash regular expression selecting version tags |
| `flake_file` | `flake.nix` | Repository-relative flake path |
| `build_target` | `.#default` | Nix installable to build |

The default `tags` mode preserves the original behavior. To package only
published GitHub Releases, configure the caller as follows:

```yaml
jobs:
  update:
    uses: ixpkgs/go-module-update-workflow/.github/workflows/update-go-module.yml@COMMIT_SHA
    with:
      discovery_mode: releases
      upstream_repository: https://github.com/example/example.git
      upstream_github_repository: example/example
      include_prereleases: false
      package_name: Example
```

Draft releases are always excluded.

## Shared discovery workflow

Updater workflows can reuse `.github/workflows/discover-upstream.yml` so that
tag-versus-release selection is implemented in one place. Call it from another
workflow in this repository with the self-repository reference:

```yaml
jobs:
  discover:
    uses: $/.github/workflows/discover-upstream.yml
    with:
      discovery_mode: ${{ inputs.discovery_mode }}
      upstream_repository: ${{ inputs.upstream_repository }}
      upstream_github_repository: ${{ inputs.upstream_github_repository }}
      tag_pattern: ${{ inputs.tag_pattern }}
      include_prereleases: ${{ inputs.include_prereleases }}
```

The workflow returns `has_updates` and `candidates_json`. The latter has a
stable, updater-independent shape:

```json
[{"tag":"v1.2.3","commit":"0123456789abcdef0123456789abcdef01234567"}]
```

The `$/` reference resolves this nested workflow from the same repository and
commit as its calling updater workflow.

## Flake contract

The selected flake file must be tracked by Git and contain exactly one assignment
of each of these forms:

```nix
tag = "v1.2.3";
commit = "0123456789abcdef0123456789abcdef01234567";
vendorHash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
```

The `vendorHash` expression must refer to the `pkgs` argument because the
workflow temporarily replaces it with `pkgs.lib.fakeHash`.
