Runs your app on a Wendy-enabled device:

1. [Selects a device](../device-selection.md)
2. [Queries the platform and architecture](./device/version.md) of this device
3. Invokes a [build](./build.md) using the target triple, and injects a [debugger](../../../debugging/) if needed
4. Uploads the artifact(s) for [Linux](../../../wendy-agent/connectivity/container-registry.md) or [macOS](../../../wendy-agent/macos/)
5. [Starts the app](./device/apps/start.md)
6. [Attaches the logs](./device/logs.md) if needed (when `--detach` is not provided)

## Compose projects

If the current directory contains a `docker-compose.yml` (or `compose.yml`) but no `wendy.json`, `wendy run` automatically runs it as a multi-service compose project. Each service is built, pushed, and started on the device in dependency order. See [Multi-Service Apps with Docker Compose](../../../apps/compose.md) for full details.

## Flags

| Flag | Description |
|------|-------------|
| `--deploy` | Build and create the container but do not start it. |
| `--detach` | Start the container but do not stream logs. |
| `--restart-unless-stopped` | Restart the container unless manually stopped. |
| `--restart-on-failure` | Restart the container on failure. |
| `--no-restart` | Do not restart the container on exit. |
| `--debug` | Enable debug logging and inject debug tooling via `WENDY_DEBUG=true`. |
| `--yes` / `-y` | Accept all device-selection prompts automatically. |
| `--build-type <type>` | Override build type detection: `docker`, `swift`, `python`, or `compose`. |
| `--prefix <dir>` | Run from a project directory other than the current working directory. |
| `--product <name>` | Swift Package Manager product to build and run (Swift projects only). |
| `--user-args <args>` | Extra arguments to pass to the container at runtime. |
