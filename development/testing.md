# Testing

## Unit and Integration Tests

All tests live in `go/` and are run with the standard Go test toolchain.

### Run all tests

```sh
cd go
make test
# equivalent to: go test ./... -v -count=1 -timeout 120s
```

### Run tests with the race detector

The CI `go-tests.yml` workflow always runs with `-race`. Mirror that locally before pushing:

```sh
make test-race
# equivalent to: go test ./... -v -count=1 -race -timeout 120s
```

### Run a specific package or test

```sh
go test ./internal/agent/services/... -v -run TestContainerService
```

### Vet and format

```sh
make vet       # go vet ./...
make fmt       # gofmt -w -s .
make lint      # golangci-lint run ./...
```

CI enforces that all files are `gofmt`-formatted. The `format` job in `go-tests.yml` runs:

```sh
UNFORMATTED=$(gofmt -l -s .)
if [ -n "$UNFORMATTED" ]; then exit 1; fi
```

Run `make fmt` before committing to avoid a CI failure.

## Integration Test File

`go/integration_test.go` at the root of the `go/` module is a Go test file that starts an in-process gRPC server backed by mock implementations (network manager, hardware discoverer, bluetooth manager, containerd client). It exercises the agent service layer end-to-end without requiring a real device or containerd installation.

These tests are included in `make test` and run on every PR via the `go-tests.yml` workflow.

## Hardware Integration Tests

End-to-end tests that deploy real container workloads to a WendyOS device are defined in `.github/ci-tests/` and orchestrated by `go/scripts/test-ci.sh`.

### Available test suites

| Name | What it tests |
|---|---|
| `swift-hello` | Basic Swift containerized deployment |
| `swift-network` | Swift with network entitlement |
| `swift-bluetooth` | Swift with Bluetooth entitlement |
| `python-hello` | Basic Python deployment |
| `python-network` | Python with network entitlement |
| `python-gpu` | Python with GPU entitlement (CUDA verification) |
| `python-bluetooth` | Python with Bluetooth entitlement |
| `python-no-network` | Verify network is blocked without entitlement |
| `python-no-bluetooth` | Verify Bluetooth is blocked without entitlement |
| `compose-hello` | docker-compose multi-service deployment with build Dockerfiles |
| `compose-images` | docker-compose multi-service deployment using public images |

Swift tests are macOS-only (skipped on the Linux runner).

### Running integration tests locally

You need a WendyOS device on your LAN and the `wendy` CLI binary.

```sh
cd go

# Build the CLI first
make build-cli

# Auto-discover a device and run all applicable tests
scripts/test-ci.sh -w ./bin/wendy

# Target a specific device
scripts/test-ci.sh -w ./bin/wendy -h wendyos-my-device.local

# Run a subset of tests
scripts/test-ci.sh -w ./bin/wendy -t python-hello -t python-network

# Show usage
scripts/test-ci.sh --help
```

The script auto-discovers devices with `wendy discover --json` if `--hostname` is not provided.

### Running integration tests in CI

The `integration-tests.yml` workflow is `workflow_dispatch` only — it runs on self-hosted runners in the `wendy-developer` group that have physical devices attached. Trigger it from the Actions tab with optional inputs:

| Input | Default | Description |
|---|---|---|
| `hostname` | _(empty)_ | Specific device hostname; empty means auto-discover |
| `tests` | _(empty)_ | Comma-separated test names; empty means all applicable |
| `jobs` | `all` | `all`, `discover`, or `integration` |
| `platform` | `all` | `all`, `macos`, or `linux` |

The macOS runner uses an SSH-to-localhost trick to obtain the Local Network TCC permission that the macOS LaunchAgent runner process lacks.

## Test Coverage

Coverage is tracked via Codecov (`codecov.yml`). Key settings:

- Coverage range for the colour scale: 50% (red) to 90% (green)
- Both project-level and patch-level gates are informational (they report but do not block the PR)
- Threshold for drop alerts: 5%
- Ignored paths: `Tests/`, `Examples/`, `.build/`, Swift test/mock/proto files

The Go test suite does not yet upload coverage to Codecov from the `go-tests.yml` workflow; that is currently handled for the Swift side. Go coverage can be measured locally:

```sh
cd go
go test ./... -coverprofile=coverage.out
go tool cover -html=coverage.out   # opens browser report
go tool cover -func=coverage.out   # per-function summary
```

## Test Infrastructure Helpers

`go/scripts/test-harness.sh` is a Bash library sourced by individual test scripts. It provides:

- Colour-coded pass/fail/skip output
- `run_test`, `run_test_expect_output`, `run_test_json`, `skip_test` functions
- `print_summary` for aggregated counts
- `discover_device` for mDNS-based device lookup
- `validate_wendy_binary` to confirm the CLI binary exists and is executable

Do not execute `test-harness.sh` directly — source it from a test script.
