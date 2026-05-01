# Contributing

## Code Owners

Two reviewers are required for all changes to the Go code: `@joannis` and `@EBro912`. The Swift companion app (`/swift/`) has a separate owner (`@konstantinbe`). EBro912 is also listed as a code owner to ensure QA sign-off before merging.

CODEOWNERS is enforced at the GitHub repository level — PRs cannot be merged without approval from the listed owners for the files changed.

## Branching Model

- `main` is the trunk branch. Every push to `main` triggers a pre-release build automatically.
- Branch names should follow the pattern `<type>/<short-description>`, e.g.:
  - `feat/bluetooth-reconnect`
  - `fix/grpc-timeout`
  - `docs/agent-setup`
  - `chore/update-dependencies`

Never commit directly to `main`.

## Commit Messages

Use the conventional commit style:

```
<type>(<scope>): <short summary>

<optional body>
```

Common types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`.

Keep the summary line under 72 characters. Use the body to explain *why*, not *what* — the diff already shows what changed.

Examples:

```
feat(agent): add mTLS server shutdown on provisioning callback

fix(cli): handle nil pointer in discover --json when no devices found

chore(deps): update containerd to v2.2.3
```

## Pull Request Workflow

1. Fork or create a branch from `main`.
2. Make your changes and ensure all checks pass locally:

   ```sh
   cd go
   make fmt        # fix formatting
   make vet        # catch static issues
   make test-race  # full test suite with race detector
   make licenses   # verify dependency licenses if you added deps
   ```

3. Open a PR against `main`. The following CI checks run automatically on every PR that touches `go/**` or `Proto/**`:
   - **Test** — `go test ./... -race -count=1 -timeout 120s`
   - **Vet** — `go vet ./...`
   - **Format Check** — `gofmt -l -s .` (fails if any file is unformatted)
4. Obtain approval from the code owners.
5. Merge. The pre-release build runs automatically after merge.

## Code Style

The project follows standard Go conventions. Key points:

- **Format**: All code must be `gofmt -s` clean. Run `make fmt` before pushing. CI will reject unformatted code.
- **Linting**: Run `make lint` (requires `golangci-lint`) to catch issues before CI does.
- **Error handling**: Return errors; do not panic in library code. Use `fmt.Errorf("context: %w", err)` to wrap errors with context.
- **Logging**: Use `go.uber.org/zap` structured logging. Pass the `*zap.Logger` down from `main`; do not use a global logger. Use `logger.Error(...)` for recoverable errors, `logger.Fatal(...)` only for startup failures where the process cannot continue.
- **Context**: Accept `context.Context` as the first parameter of every function that does I/O or can be cancelled.
- **Interfaces**: Subsystems are wired together through interfaces defined in `go/internal/agent/services/interfaces.go` (`NetworkManager`, `HardwareDiscoverer`, `BluetoothManager`, `ContainerdClient`). This keeps the service layer testable without real hardware.
- **Generated code**: Files under `go/proto/gen/` are generated. Run `make proto` to regenerate; do not edit by hand.

## Dependency Management

Add dependencies via `go get` and commit both `go.mod` and `go.sum`. After adding or upgrading dependencies:

```sh
make licenses-update   # regenerate licenses.csv
```

Commit the updated `licenses.csv` alongside your other changes. The `make licenses` check will fail in CI otherwise.

## Release Process

Releases follow a dual-track model documented in `docs/RELEASES.md`:

- **Pre-releases**: Created automatically on every `main` push with a timestamp-based tag (e.g., `2025.12.04-195108`). The Homebrew nightly formula and cask are bumped automatically.
- **Stable releases**: Manually promoted from a pre-release via the "Create Semver Release" workflow dispatch. This triggers the Homebrew stable PR, Winget update, and AUR publication.

As a contributor you do not need to manage releases. Maintainers handle promotion.

## CI Overview

| Workflow | Trigger | What runs |
|---|---|---|
| `go-tests.yml` | PR touching `go/**` or `Proto/**` | Unit tests (race), vet, format check |
| `build.yml` | Push to `main` or manual dispatch | Cross-compile all targets, package, release |
| `integration-tests.yml` | Manual dispatch | Hardware integration tests on self-hosted runners |
| `swift-tests.yml` | PR touching `swift/**` | Swift unit tests |
| `security.yml` | Schedule / push | Security scanning |

The `go-tests.yml` workflow runs on `ubuntu-latest` and installs `libasound2-dev` for the ALSA audio subsystem.
