# Wendy Agent Architecture

`wendy-agent` is a Go binary that runs on WendyOS devices and on DGX Spark. It exposes gRPC APIs consumed by the wendy CLI, manages containers through containerd, handles networking (WiFi / Bluetooth), streams OpenTelemetry telemetry, and connects devices to the Wendy Cloud.

## Startup sequence

When `wendy-agent` starts it performs the following steps in order:

1. Initialises a `zap` production logger (development mode when `WENDY_DEBUG` is set). Agent logs are tee'd into the telemetry broadcaster so they can be streamed to CLI clients.
2. Reads config from `/etc/wendy-agent` (overrideable with `WENDY_CONFIG_PATH`).
3. Applies any pending config-partition overlay (`configpartition.Apply`).
4. Commits a pending Mender A/B update (`mender-update commit`) so Mender does not roll back on the next reboot.
5. Removes stale agent binary backups older than 48 hours.
6. Ensures the NVIDIA CDI spec exists for GPU container support.
7. Initialises subsystem managers:
   - **Network manager** – wraps `nmcli` for WiFi management (skipped when `nmcli` is absent).
   - **Hardware discoverer** – probes the system for GPU, audio, video, Bluetooth, I2C, GPIO, SPI, and USB devices.
   - **Bluetooth manager** – BlueZ-backed BLE/Bluetooth management.
   - **D-Bus proxy manager** – `xdg-dbus-proxy` sandbox for Bluetooth containers (falls back to unfiltered host D-Bus access when absent).
8. Connects to containerd (default socket, overrideable with `WENDY_CONTAINERD_ADDR`).
9. Constructs all gRPC service implementations (see below).
10. Starts the container monitor (polls containerd every 15 seconds).
11. Starts background goroutines for container CPU/memory metrics and agent process metrics.
12. Decides which gRPC server(s) to start based on provisioning state (see [Security model](#security-model)).
13. Starts the embedded OCI registry (Linux only, best-effort).
14. Starts the OTEL gRPC and HTTP receivers.

## Component map

```
wendy-agent
├── gRPC servers
│   ├── Plaintext :50051      ← pre-provisioning only
│   ├── mTLS      :50052      ← post-provisioning (agentPort + 1)
│   └── OTEL gRPC :4317       ← always on
├── OTEL HTTP receiver :4318  ← always on
├── Embedded OCI registry :5000 ← Linux/containerd only
│
├── Services (registered on plaintext + mTLS servers)
│   ├── WendyAgentService       – WiFi, Bluetooth, agent update, OS update, hardware
│   ├── WendyContainerService   – container lifecycle, layers, volumes, stats
│   ├── WendyAudioService       – audio device listing and streaming
│   ├── WendyVideoService       – video device listing and streaming
│   ├── WendyProvisioningService – device enrollment with Wendy Cloud
│   └── WendyTelemetryService   – streaming OTLP logs, metrics, traces
│
├── Internal subsystems
│   ├── containerd client       – container creation, snapshots, image import
│   ├── OCI entitlements        – GPU, audio, video, Bluetooth, USB, I2C, GPIO, SPI, persist
│   ├── Container monitor       – watches running containers, restarts on crash
│   ├── Container log manager   – multiplexes container stdout/stderr to telemetry
│   ├── Network manager (nmcli) – WiFi scan, connect, disconnect, priorities
│   ├── Bluetooth manager (BlueZ) – BLE peripheral advertising, device pairing
│   ├── D-Bus proxy manager     – xdg-dbus-proxy sandboxing for BT containers
│   ├── Tunnel broker client    – persistent gRPC presence stream to Wendy Cloud
│   ├── Telemetry broadcaster   – fan-out for OTLP logs/metrics/traces
│   ├── CDI (NVIDIA)            – ensures nvidia CDI spec for GPU containers
│   └── Config partition        – applies overlay from /etc/wendy-agent
│
└── Utility subcommand
    └── wendy-agent utils open-browser <url>
```

## Security model

The agent uses two separate gRPC listeners to enforce the provisioning boundary:

| Listener | Port | TLS | Purpose |
|---|---|---|---|
| Plaintext | `50051` (default) | None | Pre-provisioning only. Shut down automatically once the device enrolls. |
| mTLS | `50052` (default, agentPort + 1) | Mutual TLS with device certificate | Post-provisioning. Carries all six services. |
| OTEL gRPC | `4317` | None | OpenTelemetry collector endpoint for container workloads. |
| OTEL HTTP | `4318` | None | OTLP/HTTP endpoint (protobuf and JSON). |
| OCI Registry | `5000` | HTTP pre-provisioning, HTTPS post-provisioning | Development container image push target. |

When `StartProvisioning` completes successfully, the agent:

1. Starts the mTLS server on `agentPort + 1`.
2. Gracefully stops the plaintext server.
3. Restarts the embedded OCI registry with HTTPS using the device certificate.
4. Starts advertising the device via Avahi mDNS using the mTLS port.
5. Starts BLE advertising and the mTLS-protected L2CAP server.
6. Connects to the Wendy Cloud tunnel broker.

Provisioning state (certificates, org/asset IDs) is persisted to `/etc/wendy-agent/provisioning.json` (mode `0600`). Individual PEM files are written alongside it for services (such as the OCI registry) that read certificates from the filesystem.

## Port summary

| Port | Protocol | Service |
|---|---|---|
| `50051` | gRPC (plaintext) | Agent API – pre-provisioning |
| `50052` | gRPC (mTLS) | Agent API – post-provisioning |
| `4317` | gRPC | OTLP log/metric/trace receiver |
| `4318` | HTTP | OTLP/HTTP receiver |
| `5000` | HTTP or HTTPS | Embedded OCI registry |

All ports can be overridden with environment variables (`WENDY_AGENT_PORT`, `WENDY_OTEL_PORT`, `WENDY_OTEL_HTTP_PORT`, `WENDY_REGISTRY_ADDR`).

## Cloud connectivity

After provisioning, the tunnel broker client (`services.TunnelBrokerClient`) opens a `RegisterPresence` streaming RPC to the Wendy Cloud broker. The broker URL defaults to `<cloudHost>:50052`; when the cloud host uses port `443` the same address is used directly. The client reconnects with exponential backoff (1 s – 5 min) on failure.

## mDNS advertisement

On provisioned devices the agent uses Avahi to advertise `_wendyos._udp` on the mTLS port. Pre-provisioning the Avahi service file (installed at `/etc/avahi/services/wendy-agent.service`) advertises port `50051`. The CLI discovers local devices via this mDNS service type.

## OCI entitlements

Container specs are built by the agent via the `oci` package. The `ApplyEntitlements` function maps `wendy.json` entitlement declarations to OCI spec modifications:

| Entitlement | Effect |
|---|---|
| `gpu` | Mounts NVIDIA CDI devices, adds NVIDIA capabilities |
| `network` | Sets network mode (`host` or `none`) |
| `audio` | Bind-mounts `/dev/snd`, adds audio group (GID 29) |
| `video` / `camera` | Bind-mounts V4L2 devices, adds video group (GID 44) |
| `bluetooth` | Mounts filtered D-Bus socket (via `xdg-dbus-proxy`) or raw host socket |
| `usb` | Bind-mounts `/dev/bus/usb` |
| `i2c` | Bind-mounts specified I2C device |
| `gpio` | Bind-mounts specified GPIO device |
| `spi` | Bind-mounts SPI devices |
| `persist` | Mounts named volume from `/var/lib/wendy/volumes/` |
| `input` | Bind-mounts `/dev/input/`, adds input group (GID 105) |
