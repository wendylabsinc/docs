A tool that uploads, removes and promotes (nightly -> release) builds for [WendyOS](../wendyos/) and [Wendy Lite](../wendy-lite/).

The publisher source lives at `tools/publisher/` in [wendylabsinc/wendyos-builder](https://github.com/wendylabsinc/wendyos-builder), versioned and CI-gated alongside the build workflow that drives it. The original standalone repository is [wendylabsinc/wendy-os-publisher](https://github.com/wendylabsinc/wendy-os-publisher).

## CI integration

### Nightly builds

The nightly pipeline runs all device builds in parallel. Each build job uploads image files to GCS and emits a manifest-entry artifact; it does **not** write to any shared manifest. The `publish` job collects all manifest-entry artifacts and is the sole writer of device and master manifests in GCS. This serialised design prevents concurrent read-modify-write races on shared manifest files.

The `publish` job:

- Applies device manifest entries one at a time from the collected artifacts.
- Updates the master manifest once per device that actually published.
- Creates a timestamped GitHub pre-release (`nightly-YYYYMMDDTHHMMSS`) and sends a Discord announcement **only when at least one device build produced artifacts**. If every build fails, neither the pre-release nor the Discord notification fires.
- The GitHub pre-release is tagged `nightly-YYYYMMDDTHHMMSS`. Release notes are auto-generated from PRs and commits (`--generate-notes`). The 7 most recent nightly pre-releases are retained; older ones are deleted (git tags are kept forever for traceability).
- The Discord announcement is sent via a single `curl`/`jq` step in the `publish` job, not per-device.

Concurrent workflow runs for the same ref are serialised: PR runs cancel superseded builds; main/tag runs queue so a publish in progress is never killed halfway.

### Release builds

The release pipeline triggers on tags matching `MAJOR.MINOR.PATCH` (no `v` prefix — git tags, GitHub releases, and GCS manifests all use the same plain version string). The tag must match `DISTRO_VERSION` in `conf/distro/wendyos.conf`; mismatches are caught at both the tag-push step and the promote workflow before any tag is created.

The `publish` job runs for release builds as well as nightly builds. A dedicated Discord announcement step in `promote.yml` sends the release notification **only after** the release is successfully created. The Discord webhook is invoked once, not per-device.

### Promoting a nightly to a release

Use the `promote` workflow (`promote.yml`). Inputs:

- **`release_tag`** — the version to create, e.g. `1.2.3` (plain `MAJOR.MINOR.PATCH`; a leading `v` is stripped automatically).
- **`source_nightly`** — the `nightly-YYYYMMDDTHHMMSS` tag to promote from. Leave empty to use the most recent `nightly-*` tag automatically.

The workflow validates that the release version matches `DISTRO_VERSION` at the nightly commit before creating any tag, and is idempotent — re-running after a partial failure is safe.
