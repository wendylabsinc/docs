# `wendy watch`

Watches the project directory and automatically rebuilds and redeploys the app to the target device whenever a file changes.

## Usage

```sh
wendy watch [flags]
```

## Description

`wendy watch` is the inner-loop command for iterative development. It runs the same build + chunk-diff deploy pipeline as `wendy run`, but in a loop: every time you save a file, the app is rebuilt and the running container is replaced on the device. Deploys run in the background (detached); use `wendy device logs` to tail the app's output.

**What triggers a redeploy:**

- Any file write under the project directory (recursive).
- Saves are debounced (default 400 ms) so rapid bursts from formatters or IDEs produce a single deploy.
- An in-flight deploy is cancelled and restarted if a newer change arrives before it completes.

**What is ignored:**

- Hidden directories (`.git`, `.build`, etc.) and common dependency trees (`node_modules`, `.venv`, `vendor`).
- Editor temp files (`.swp`, `~`, `#`).

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--build-type <type>` | auto | Override build type: `docker`, `swift`, or `python`. Useful when multiple build markers coexist. |
| `--builder <name>` | auto | Force a specific image builder for Dockerfile/Containerfile builds: `docker` or `apple-container`. |
| `--dockerfile <file>` | — | Dockerfile to use (e.g. `Dockerfile.prod`). |
| `--prefix <dir>` | `.` | Project directory to watch instead of the current working directory. |
| `--product <name>` | — | Swift Package Manager product to build and run. |
| `--service <name>` | — | Build and run only the named service and its dependencies (multi-service projects). |
| `--user-args <args>` | — | Extra arguments to pass to the container at runtime. |
| `--restart-unless-stopped` | `false` | Restart the container unless manually stopped. |
| `--restart-on-failure` | `false` | Restart the container on failure. |
| `--debounce <ms>` | `400` | Quiet period in milliseconds after the last change before redeploying. |
| `--debug` | `false` | Enable debug logging. |

[Device selection flags](../device-selection.md) (`--device`, `--yes`) are also accepted.

## Examples

Watch the current directory and redeploy to the default device:

```sh
wendy watch
```

Watch a subdirectory:

```sh
wendy watch --prefix apps/my-service
```

Use Apple Container as the builder (Apple silicon, no Docker required):

```sh
wendy watch --builder apple-container
```

Increase the debounce window to 1 second:

```sh
wendy watch --debounce 1000
```

## See also

- [`wendy run`](./run.md) — one-shot build and deploy
- [`wendy device logs`](./device/logs.md) — tail logs from the running container
