# Threat Model

## OTEL receivers — localhost-only binding (WDY-1097, WDY-1100)

**Fixed in:** PR #597

### Threat

Prior to PR #597, the wendy-agent bound both OTEL receivers to `[::]` (all interfaces), exposing ports 4317 (gRPC) and 4318 (HTTP) to any host on the network. Both endpoints are unauthenticated. This created two related attack surfaces:

| ID | Threat |
|----|--------|
| WDY-1097 | Any network peer could inject arbitrary telemetry data into the agent (metrics, traces, logs) — potentially poisoning dashboards, alerting pipelines, or audit trails. |
| WDY-1100 | Any network peer could submit unbounded telemetry payloads, causing resource exhaustion (CPU, memory, disk) on the device. |

### Fix

Both receivers are now bound to `127.0.0.1` only:

```
// Before
net.Listen("tcp", "[::]:"+otelPort)
net.Listen("tcp", "[::]:"+otelHTTPPort)

// After
net.Listen("tcp", "127.0.0.1:"+otelPort)
net.Listen("tcp", "127.0.0.1:"+otelHTTPPort)
```

Only processes running locally on the device can reach the OTEL endpoints. Network peers cannot connect regardless of firewall configuration.

### Verification

- **Unit:** `TestOTELLocalhostBindProperty` in `go/integration_test.go` verifies that a `127.0.0.1`-bound listener is not reachable via a non-loopback IP.
- **Hardware CI:** `scripts/test-ci.sh -t otel-localhost-only` uses `nc` to assert ports 4317 and 4318 are not reachable from outside the device.

See [wendy-agent/otel.md](../wendy-agent/otel.md) for full OTEL receiver documentation.
