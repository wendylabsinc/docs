# OpenTelemetry

The Wendy agent includes a built-in OpenTelemetry (OTel) collector. It accepts logs, metrics, and traces from apps running on the device, and can forward them to Wendy Cloud for centralized visibility.

## Receiving telemetry from your app

The collector listens on loopback-only endpoints accessible to any container with network access:

| Transport | Address |
|-----------|---------|
| gRPC | `localhost:4317` |
| HTTP | `localhost:4318` |

Use the standard OTLP exporter from your language's OTel SDK and point it at one of these addresses. Any `stdout` output is also captured and converted into OTel log records, with the app's ID attached as the originating service.

## Cloud export

When the device is enrolled in Wendy Cloud and a cloud session is active, the agent forwards collected telemetry to the cloud over OTLP gRPC. Telemetry is sanitized before export — attribute values that exceed size limits or contain reserved keys are dropped with a warning, preventing unbounded payloads from reaching the cloud.

Once forwarded, telemetry is visible to any authorized client via [`wendy device logs`](../clients/wendy-cli/commands/device/logs.md), [`wendy device dashboard`](../clients/wendy-cli/commands/device/dashboard.md), and [`wendy device telemetry-stream`](../clients/wendy-cli/commands/device/telemetry-stream.md).

## Local-only access

When no cloud session is configured, telemetry remains local. You can still access it directly over the CLI while connected to the device.
