# Tailscale Patch Build Repository

## Purpose

This repository is the **patch source and build/release entry point** for a small set of custom Tailscale features. Its GitHub Actions workflow checks out an official Tailscale release tag, applies the patches stored here, runs targeted tests, builds binaries, and publishes the resulting assets to this repository's GitHub Releases.

The patches were developed and tested in the separate Tailscale fork (`3rd-repo/tailscale`, branch `feature/custom`). The fork remains useful as the editable development source and retains the custom code in context with the upstream repository. This repository does not clone or depend on that fork during builds: the workflow uses only the selected upstream tag and the patch files checked into this repository.

In short:

- **Tailscale fork**: develop, adapt, review, and test custom source changes against upstream.
- **This repository**: maintain exported patches and run reproducible build/release jobs.
- **Tailscale upstream**: the clean source tree selected for each build.

## Contents

| Path | Purpose |
| --- | --- |
| `patches/0001-force-derp.patch` | Runtime control to force peer traffic through DERP, including tests. |
| `patches/0002-default-no-logs-no-support.patch` | Enable `-no-logs-no-support` by default. |
| `patches/0003-show-tailscaled-subcommands.patch` | Show selected `tailscaled` subcommands and debug usage in help output. |
| `patches/0004-system-service-install-and-args.patch` | Linux systemd installation and custom service argument support on supported platforms. |
| `BASE_COMMIT` | Upstream commit used as the base when the current patch set was exported. It is provenance information, not the release tag selected at Action runtime. |
| `SOURCE_COMMIT` | Tailscale fork commit from which the current patch set was exported. |
| `.github/workflows/release-custom.yml` | Manual build, test, and release workflow executed in this repository. |

Patches are applied in lexicographic filename order. Keep the numeric prefixes and update this README if features, ordering, or responsibilities change. Repository documentation and workflows are not applied to the Tailscale source tree.

## Features currently carried

1. **Force DERP** — add `tailscale debug force-derp [true|false]` to query or toggle force-relay mode. The associated magicsock behavior disables direct UDP paths while enabled.
2. **No logs/support by default** — default `tailscaled` to `-no-logs-no-support=true`; an explicit flag value can override it.
3. **Visible daemon help** — document the supported `tailscaled` service and debug subcommands in help output, and show debug usage when invoked without arguments.
4. **System service installation** — install/uninstall a Linux systemd service and persist supplied daemon arguments for platform service installers.

The workflow also builds `derper` for Linux amd64. That is build configuration, not a source feature patch.

## Build and release

The workflow is `.github/workflows/release-custom.yml` and runs in this repository. To use it:

1. Push the desired patch set and workflow changes to this repository's default branch.
2. In GitHub, open **Actions → Build Custom Tailscale → Run workflow**.
3. Enter an official Tailscale version tag, for example `v1.104.6`. To label the custom release separately, enter `v1.104.6-custom`; the workflow checks out upstream tag `v1.104.6` and uses the supplied value as the artifact/release version label.
4. The workflow fetches the official tag, applies every `patches/*.patch` file in order, and runs these targeted tests before building:

   ```text
   go test ./cmd/tailscale/... ./cmd/tailscaled/... ./ipn/localapi/... ./ipn/ipnlocal/... ./wgengine/magicsock/...
   ```

5. Build jobs produce `tailscale` and `tailscaled` for Linux amd64/arm64, macOS arm64, and Windows amd64, plus `derper` for Linux amd64. The release job adds `sha256sums.txt` and publishes assets to this repository's GitHub Releases.

The workflow requires repository Actions permissions that allow `GITHUB_TOKEN` to write releases/content. A successful patch application is not enough to establish compatibility: tests and builds must also succeed. In particular, a patch may apply cleanly to one upstream tag but conflict with another, or compile while still having a behavioral regression.

## Why a patch can apply to a newer tag but conflict during rebase

A successful Action run against a newer release tag means the patch's recorded context matched that tag; it does not mean Git can replay the original fork commits onto the current `main` branch. `git rebase` replays each historical commit and uses that commit's original parent/context. The patch workflow applies a final diff to a selected tag and can therefore succeed even when replaying an older development commit conflicts with upstream edits to the same functions.

Compatibility is specific to the chosen source version and patch. Record the exact upstream tag used for each release and rerun the workflow when changing tags or patches.

## Patch maintenance workflow

### 1. Develop against current upstream in the fork

Keep `main` in the Tailscale fork as the upstream-synchronized branch. Develop custom code on `feature/custom`, rebased onto the latest `origin/main` when practical. Resolve semantic conflicts by adapting the feature to the new upstream implementation; do not merely force the old patch through. Run relevant tests and cross-platform builds before exporting changes.

The fork's own release Action is not the publishing path. The authoritative build and release process is this repository's workflow. The fork's `main` may retain its Action file if desired, but builds are triggered from this patch repository.

### 2. Export one patch per feature

Use the fork's current upstream base and reviewed feature source commit. Export each feature's **actual feature file list** separately; do not blindly split a combined diff when features overlap or share files. Ensure the patch set, applied in numeric order to the chosen base, recreates the intended custom source changes.

For example, from the root of the Tailscale fork:

```bash
BASE=$(git merge-base origin/main feature/custom)
SOURCE=$(git rev-parse feature/custom)
PATCH_DIR=../tailscale-patch/patches
mkdir -p "$PATCH_DIR"

git diff --binary --full-index "$BASE" "$SOURCE" -- \
  cmd/tailscale/cli/debug.go cmd/tailscale/cli/debug_forcederp.go \
  cmd/tailscale/cli/debug_forcederp_test.go ipn/ipnlocal/forcederp.go \
  ipn/localapi/debug.go ipn/localapi/debug_forcederp.go \
  ipn/localapi/debug_forcederp_test.go wgengine/magicsock/endpoint.go \
  wgengine/magicsock/forcederp.go wgengine/magicsock/forcederp_test.go \
  wgengine/magicsock/magicsock.go \
  > "$PATCH_DIR/0001-force-derp.patch"

git diff --binary --full-index "$BASE" "$SOURCE" -- cmd/tailscaled/tailscaled.go \
  > "$PATCH_DIR/0002-default-no-logs-no-support.patch"
# For files shared by independent features, export only the intended hunks;
# inspect and test the resulting patches instead of duplicating whole-file diffs.
git diff --binary --full-index "$BASE" "$SOURCE" -- <service-installer files> \
  > "$PATCH_DIR/0004-system-service-install-and-args.patch"
printf '%s\n' "$BASE" > ../tailscale-patch/BASE_COMMIT
printf '%s\n' "$SOURCE" > ../tailscale-patch/SOURCE_COMMIT"},{
printf '%s\n' "$BASE" > ../tailscale-patch/BASE_COMMIT
printf '%s\n' "$SOURCE" > ../tailscale-patch/SOURCE_COMMIT
```

Use stable, descriptive filenames and apply-order numbers. Avoid duplicate hunks across patches. If a source file contains independent features, construct or review the patches carefully so each contains only its feature's changes. Update the feature table and any instructions here when the patch set changes.

### 3. Validate the exported patch series

Start from a clean worktree/archive of `BASE_COMMIT` (or another explicitly selected target upstream tag). Run the loop from the Tailscale source root and provide the patch repository path:

```bash
PATCH_DIR=/path/to/tailscale-patch/patches
for patch in "$PATCH_DIR"/*.patch; do
  git apply --check "$patch"
  git apply "$patch"
done
```

Then inspect the resulting diff, run the same tests as the Action, and cross-compile the supported targets. Also run the actual GitHub Action against the intended official release tag before relying on it for publication. Update `BASE_COMMIT` and `SOURCE_COMMIT` whenever a new patch export becomes authoritative.

### 4. Commit and publish from this repository

Commit the refreshed patch files, provenance files, README, and workflow changes to this repository. Run the Action from its default branch and record the upstream tag and resulting release. Do not edit the applied source only inside the workflow; make fixes in the fork's development source, then re-export and validate the patches here.

## Long-term maintenance notes

- `BASE_COMMIT` identifies the upstream baseline for the exported patch set; it is not necessarily the tag built by a release run.
- `SOURCE_COMMIT` identifies the source revision from the Tailscale fork used to export the patches. Keep it reachable in that fork or otherwise preserve the commit/repository history for traceability.
- A source patch that applies to the base commit may not apply to a newer official release. Rebase/adapt in the fork, regenerate the patch, and test the actual release tag.
- Keep custom features isolated, documented, and tested. Include tests with code changes wherever possible.
- Patch application order is part of the build input. Renaming prefixes or introducing dependencies requires updating and validating the full series.
- The Action currently uses `actions/checkout@v4`, `actions/setup-go@v5`, and `actions/upload-artifact@v4`; periodically review workflow dependencies and GitHub Actions permissions.
- Releases are published to this repository, not to the Tailscale fork or upstream project.
