A tool that uploads, removes and promotes (nightly -> release) builds for [WendyOS](../wendyos/) and [Wendy Lite](../wendy-lite/).

https://github.com/wendylabsinc/wendy-os-publisher

Wendy OS Publisher is integrated into WendyOS CI on Github actions, where it uploads as nightly.
You can manually run this tool to promote a build to release.

## CI integration

### Nightly builds

The nightly pipeline runs all device builds in parallel. The `publish` job:

- Always updates the master manifest (runs even if some builds are cancelled).
- Creates a GitHub pre-release and sends a Discord announcement **only when all device builds succeed**. If any build fails, neither the pre-release nor the Discord notification fires.
- The GitHub pre-release is tagged `nightly` and also tagged `${VERSION}-nightly`. Release notes are auto-generated from PRs and commits since the last `vX.Y.Z` tag (`--generate-notes`).
- The Discord announcement is sent via a single `curl`/`jq` step in the `publish` job, not per-device.

### Release builds

The release pipeline triggers on `v*` tags. After all device builds complete:

- A GitHub release is created (idempotent — skipped if the tag already has a release).
- A Discord announcement is sent **only when the release is newly created**. Re-running the job does not send a duplicate notification.
- The Discord announcement step fails loudly on HTTP errors (`curl -fsS`).
- As with nightly, the Discord webhook is invoked once from the `create-github-release` job, not per-device.
