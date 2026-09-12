# go-module-update-workflow

A reusable GitHub Actions workflow for Nix flakes that package tagged Go
releases with `buildGoModule`.

For every upstream tag that is not already present in the caller repository,
the workflow:

1. updates `tag` and `commit` in the caller's flake;
2. asks Nix for the new `vendorHash`;
3. verifies the build;
4. creates a commit and matching Git tag; and
5. atomically pushes the commit and tag to the caller's default branch.

Each successfully packaged release is pushed independently, in version order.

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
| `tag_pattern` | semantic `v` tags | Bash regular expression selecting release tags |
| `flake_file` | `flake.nix` | Repository-relative flake path |
| `build_target` | `.#default` | Nix installable to build |

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
