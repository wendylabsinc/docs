# OpenTelemetry (OTEL) Receivers

The wendy-agent embeds two OpenTelemetry receivers so that apps running on the device can emit telemetry (logs, metrics, traces) without any additional sidecar or collector:

| Receiver | Protocol | Port | Bind address |
|----------|----------|------|--------------|
| gRPC     | OTLP/gRPC | 4317 | `127.0.0.1`  |
| HTTP     | OTLP/HTTP | 4318 | `127.0.0.1`  |

## Security: localhost-only binding

Both receivers are bound to `127.0.0.1` only. They are **not** reachable from the network — only processes running locally on the device can submit telemetry. This prevents:

- **Unauthenticated data injection** from any network source (WDY-1097)
- **Resource exhaustion** via unauthenticated endpoints exposed to the network (WDY-1100)

If you need to forward telemetry from a remote source, send it through the wendy-agent gRPC API instead.

## Consuming telemetry

Apps configured with a network entitlement can submit OTLP data to the receivers directly:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4317   # gRPC
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318   # HTTP
```

The agent fans out all received telemetry to connected clients via its broadcaster. The CLI commands [`wendy device logs`](../clients/wendy-cli/commands/device/logs.md), [`wendy device dashboard`](../clients/wendy-cli/commands/device/dashboard.md), and [`wendy device telemetry-stream`](../clients/wendy-cli/commands/device/telemetry-stream.md) all consume this stream.

## Supported signals

| Signal  | gRPC service                           | HTTP path            |
|---------|----------------------------------------|----------------------|
| Metrics | `opentelemetry.proto.MetricsService`   | `/v1/metrics`        |
| Traces  | `opentelemetry.proto.TraceService`     | `/v1/traces`         |

## Container apps

Apps running in containers should configure their OTLP exporter to point at the host's loopback address. When using `network_mode: host` this is simply `127.0.0.1`. When using an isolated network namespace, use the gateway address for the container network (typically the Docker bridge IP).

## Testing

### Unit test

`TestOTELLocalhostBindProperty` (in `go/integration_test.go`) verifies the security property of the localhost-only binding:

1. Finds a non-loopback IPv4 interface on the test machine. Skips if none exists (e.g. a stripped-down CI container with only `lo`).
2. Binds a `127.0.0.1`-only TCP listener, mirroring what the OTEL receivers do.
3. Asserts that a connection from `127.0.0.1` succeeds.
4. Asserts that a connection via the external interface IP **fails**, confirming the listener is not reachable from the network.

Run with:

```sh
go test .
```

### Hardware CI test

The `otel-localhost-only` test in `scripts/test-ci.sh` uses `nc` to assert that ports 4317 and 4318 are not reachable from the developer machine:

```sh
scripts/test-ci.sh -t otel-localhost-only -h <device-hostname>
```

Two sub-checks are run:

| Sub-test | What it verifies |
|----------|-----------------|
| `otel-localhost-only (gRPC 4317 not reachable)` | Port 4317 is not reachable from the network |
| `otel-localhost-only (HTTP 4318 not reachable)` | Port 4318 is not reachable from the network |
