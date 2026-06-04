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

The release pipeline is triggered by pushing a `v*` tag. The promote workflow:

1. Pushes the `v*` tag to the repository.
2. Immediately creates a GitHub release (idempotent — skipped if the tag already has a release) and sends a Discord announcement.
3. The tag push auto-triggers `build.yml`, which builds all device images in parallel and uploads them as the release version to the manifest.

The `publish` job (master manifest update) is skipped for tag-triggered builds — the per-device upload steps write directly to the release manifest without a separate publish step.

The Discord announcement fires only when a new GitHub release is created (not on an already-existing release tag). It uses `jq` to build the webhook payload and is sent inline from the promote job, immediately after the tag is pushed.
