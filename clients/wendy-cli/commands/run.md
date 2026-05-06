Runs your app on a Wendy-enabled device:

1. [Selects a device](../device-selection.md)
2. [Queries the platform and architecture](./device/version.md) of this device
3. Invokes a [build](./build.md) using the target triple, and injects a [debugger](../../../debugging/) if needed
4. Uploads the artifact(s) for [Linux](../../../wendy-agent/connectivity/container-registry.md) or [macOS](../../../wendy-agent/macos/)
5. [Starts the app](./device/apps/start.md)
6. [Attaches the logs](./device/logs.md) if needed (when `--detach` is not provided)

## Compose projects

If the current directory contains a `docker-compose.yml` (or `compose.yml`) but no `wendy.json`, `wendy run` automatically runs it as a multi-service compose project. Each service is built, pushed, and started on the device in dependency order. See [Multi-Service Apps with Docker Compose](../../../apps/compose.md) for full details.

## Swift Package Manager projects (macOS)

When running a Swift Package Manager project on a macOS target, `wendy run`:

1. Builds the project with `swift build -c release` (or `-c debug` when `--debug` is passed).
2. Resolves the build products directory via `swift build --show-bin-path`.
3. Syncs the compiled binary to the device.
4. Automatically syncs any sibling `.bundle` and `.resources` directories found in the build products directory alongside the binary, so SwiftPM resource bundles are available at runtime.
5. Syncs `sandbox.sb` from the project root if present, and any additional files declared under `files` in `wendy.json`.

## Flags

| Flag | Description |
|------|-------------|
| `--deploy` | Build and create the container but do not start it. |
| `--detach` | Start the container but do not stream logs. |
| `--restart-unless-stopped` | Restart the container unless manually stopped. |
| `--restart-on-failure` | Restart the container on failure. |
| `--no-restart` | Do not restart the container on exit. |
| `--debug` | Enable debug logging and inject debug tooling via `WENDY_DEBUG=true`. For SwiftPM projects, builds with `-c debug` instead of `-c release`. |
| `--yes` / `-y` | Accept all device-selection prompts automatically. |
| `--build-type <type>` | Override build type detection: `docker`, `swift`, `python`, or `compose`. |
| `--prefix <dir>` | Run from a project directory other than the current working directory. |
| `--product <name>` | Swift Package Manager product to build and run (Swift projects only). |
| `--user-args <args>` | Extra arguments to pass to the container at runtime. |

## postStart hook process lifetime

When a `postStart.cli` hook is configured in `wendy.json`, `wendy run` starts the hook command after the app is ready and tracks its process lifetime.

On **Windows**, the entire process tree spawned by the hook — including grandchildren started via `start /B` — is terminated when `wendy run` exits or is interrupted. This is implemented using a Windows Job Object with `KILL_ON_JOB_CLOSE`; closing the job handle causes the kernel to terminate every process assigned to it. If Job Object creation is unavailable, `wendy run` falls back to `taskkill /T /F`, which terminates the direct child and its descendants as long as the parent process is still alive.

On **Unix**, the default shell process-group cleanup is sufficient; no additional termination logic is applied.
