# Integration Tests

The Wendy CI integration test suite runs end-to-end tests against real devices (or device emulators) to verify that `wendy run` correctly builds, deploys, and operates apps.

Tests live under `.github/ci-tests/` and are executed by `go/scripts/test-ci.sh`.

## Available tests

| Test name | Language | What it verifies |
|-----------|----------|-----------------|
| `swift-hello` | Swift | Basic Swift app builds and runs (macOS only) |
| `swift-network` | Swift | Swift app with `network` entitlement (macOS only) |
| `swift-bluetooth` | Swift | Swift app with `bluetooth` entitlement (macOS only) |
| `python-hello` | Python | Basic Python app builds and runs |
| `python-network` | Python | Python app with `network` entitlement |
| `python-gpu` | Python | Python app with `gpu` entitlement |
| `python-bluetooth` | Python | Python app with `bluetooth` entitlement |
| `python-no-network` | Python | Verifies network is blocked **without** `network` entitlement |
| `python-no-bluetooth` | Python | Verifies bluetooth is blocked **without** `bluetooth` entitlement |
| `python-no-ptrace` | Python | Verifies `ptrace` syscall is blocked by the default seccomp profile (WDY-1099) |
| `python-no-unshare` | Python | Verifies `unshare` syscall is blocked by the default seccomp profile (WDY-1099) |
| `compose-hello` | Compose | `docker-compose` multi-service deployment with `build:` Dockerfiles |
| `compose-images` | Compose | `docker-compose` multi-service deployment using public images |
| `otel-localhost-only` | Python | Verifies OTEL receivers (ports 4317/4318) are not reachable from the network |

> **Note:** Swift tests (`swift-*`) are skipped automatically on Linux runners because Swift container builds require macOS.

## Running the suite locally

```sh
cd go
scripts/test-ci.sh -h <device-hostname>
```

Run a specific subset of tests with one or more `-t` flags:

```sh
scripts/test-ci.sh -h <device> -t python-hello -t python-no-ptrace
```

Run all tests:

```sh
scripts/test-ci.sh -h <device>
```

## Seccomp tests (WDY-1099)

`python-no-ptrace` and `python-no-unshare` are **negative tests** — they assert that certain dangerous syscalls are denied inside Wendy app containers by the default seccomp profile applied by `wendy-agent`.

### `python-no-ptrace`

Calls `ptrace(PTRACE_TRACEME, 0, 0, 0)` via `ctypes` and expects the call to be denied (`EPERM`/`EACCES`). A successful `ptrace` call would indicate the seccomp profile was not applied.

### `python-no-unshare`

Calls `unshare(CLONE_NEWUSER)` via `ctypes` and expects the call to fail. `unshare(CLONE_NEWUSER)` is the first step in a common unprivileged container-escape chain. Denial confirms the seccomp profile blocks the `unshare` syscall entirely.

Both tests exit `0` on a correct denial and `1` if the syscall unexpectedly succeeds.

## CI matrix

The workflow (`integration-tests.yml`) runs tests in parallel across available devices, one `docker buildx` builder instance per device:

- **macOS runners:** full suite including `swift-*` tests.
- **Linux runners:** Python and compose tests only; `swift-*` tests are skipped automatically.

Each parallel job uses a dedicated buildx builder (`wendy-0`, `wendy-1`, …) named after its index in the device list. The builder is removed after the job completes regardless of pass/fail status.
