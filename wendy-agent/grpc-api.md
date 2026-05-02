# Wendy Agent gRPC API

All proto files live under `Proto/wendy/agent/services/v1/` in the `wendy-agent` repository. Generated Go bindings are in `go/proto/gen/agentpb/`.

## Ports

| Port | Transport | When active |
|---|---|---|
| `50051` | Plaintext gRPC | Pre-provisioning only. Shut down automatically once the device enrolls. |
| `50052` | Mutual-TLS gRPC | Post-provisioning. All seven services are registered here. |

`50051` is the default; override with `WENDY_AGENT_PORT`. The mTLS port is always `WENDY_AGENT_PORT + 1`.

---

## WendyAgentService

`Proto/wendy/agent/services/v1/wendy_agent_v1_service.proto`

General device management: agent version, WiFi, Bluetooth, hardware, agent self-update, and OS update.

| Method | Type | Description |
|---|---|---|
| `GetAgentVersion` | Unary | Returns agent version, OS, CPU architecture, WendyOS version, device type, GPU info (vendor, JetPack, CUDA), storage medium, and a `featureset` list (`gpu`, `audio`, `bluetooth`, `video`, `camera`, `mender`). |
| `UpdateAgent` | Bidi-stream | Receives the new binary in chunks, then a control message with expected SHA-256. Verifies the hash, atomically replaces the binary, and exits so systemd restarts the agent. |
| `UpdateOS` | Server-stream | Downloads a Mender artifact from a URL, runs `mender-update install`, streams progress phases (`downloading`, `installing`, `finalizing`) and percent, then reboots. |
| `RunContainer` | Bidi-stream | **Deprecated.** Use `WendyContainerService`. |
| `ListWiFiNetworks` | Unary | Returns visible SSIDs with signal strength, security type, known/connected flags, and RSSI. |
| `GetWiFiStatus` | Unary | Returns whether the device is connected and the current SSID. |
| `ConnectToWiFi` | Unary | Connects to a WiFi network by SSID and password; supports hidden networks and security type hints. |
| `DisconnectWiFi` | Unary | Disconnects from the current WiFi network. |
| `ListKnownWiFiNetworks` | Unary | Lists saved NetworkManager profiles (SSID, UUID, priority, security). |
| `SetWiFiNetworkPriority` | Unary | Sets the autoconnect priority of a saved network. |
| `ReorderKnownWiFiNetworks` | Unary | Bulk-reorders saved networks by priority. |
| `ForgetWiFiNetwork` | Unary | Removes a saved NetworkManager profile. |
| `ListHardwareCapabilities` | Unary | Returns hardware device list (GPU, USB, I2C, GPIO, camera, etc.) with paths and properties. Optional `category_filter`. |
| `ScanBluetoothPeripherals` | Bidi-stream | Streams discovered Bluetooth devices (name, address, RSSI, paired/connected/trusted flags) until the client closes. |
| `ConnectBluetoothPeripheral` | Unary | Connects to a Bluetooth device by address; optionally pairs and trusts. |
| `DisconnectBluetoothPeripheral` | Unary | Disconnects a Bluetooth device by address. |
| `ForgetBluetoothPeripheral` | Unary | Removes a paired Bluetooth device by address. |

WiFi management delegates to `nmcli` via `NMCLINetworkManager`. All WiFi methods return `codes.Unavailable` when `nmcli` is not installed.

---

## WendyContainerService

`Proto/wendy/agent/services/v1/wendy_agent_v1_container_service.proto`

Full container lifecycle management backed by containerd.

| Method | Type | Description |
|---|---|---|
| `ListLayers` | Server-stream | Lists layer digests and sizes already present in containerd's content store. |
| `WriteLayer` | Client-stream | Uploads a single OCI layer (identified by digest). |
| `CreateContainer` | Unary | Creates a container from a previously written image. |
| `CreateContainerWithProgress` | Server-stream | Like `CreateContainer` but streams unpacking progress (phase, layer index, size, snapshot-reuse flag). |
| `RunContainer` | Server-stream | Runs a container from uploaded layers; streams `Started` then stdout/stderr. |
| `StartContainer` | Server-stream | Starts a previously created container by name; streams stdout/stderr. |
| `AttachContainer` | Bidi-stream | Attaches to a running container (sends stdin, receives stdout/stderr). |
| `StopContainer` | Unary | Stops a named container. |
| `DeleteContainer` | Unary | Deletes a container and optionally its image and persistent volumes. |
| `ListContainers` | Server-stream | Streams all known containers with name, version, and running state. |
| `ListVolumes` | Unary | Lists persistent volumes (`/var/lib/wendy/volumes/`) with name, path, size, creation time, and apps that use them. |
| `RemoveVolume` | Unary | Removes a named persistent volume. |
| `ListContainerStats` | Unary | Returns current memory and storage usage for all containers. |

---

## WendyAudioService

`Proto/wendy/agent/services/v1/wendy_agent_v1_audio_service.proto`

PipeWire/WirePlumber-backed audio device management and streaming.

| Method | Type | Description |
|---|---|---|
| `ListAudioDevices` | Unary | Lists input (microphone) and output (speaker) devices with PipeWire node ID, name, description, and default flag. Optional type filter. |
| `SetDefaultAudioDevice` | Unary | Sets the default input or output device by node ID. |
| `StreamAudioLevels` | Server-stream | Streams real-time peak and RMS dB levels for a device at 1–60 Hz. |
| `StreamAudio` | Server-stream | Streams raw s16le PCM chunks from a microphone at a configurable sample rate and channel count. |

---

## WendyVideoService

`Proto/wendy/agent/services/v1/wendy_agent_v1_video_service.proto`

V4L2 video device listing and streaming.

| Method | Type | Description |
|---|---|---|
| `ListVideoDevices` | Unary | Lists `/dev/videoN` devices with numeric ID, human-readable name, and path. |
| `StreamVideo` | Server-stream | Streams encoded video frames (H.264 annexb or VP8/WebM) from a V4L2 device. Width, height, and framerate default to the device's native values when set to `0`. |

---

## WendyProvisioningService

`Proto/wendy/agent/services/v1/wendy_agent_v1_provisioning_service.proto`

Device enrollment with the Wendy Cloud certificate authority.

| Method | Type | Description |
|---|---|---|
| `IsProvisioned` | Unary | Returns `ProvisionedResponse` (cloud host, org ID, asset ID) when enrolled, or `NotProvisionedResponse`. |
| `StartProvisioning` | Unary | Generates a CSR for the device key, exchanges it with the Wendy Cloud CA using an enrollment token, stores the issued certificate, and triggers the mTLS server startup and plaintext shutdown. |

Provisioning state is persisted to `/etc/wendy-agent/provisioning.json`. The device private key is stored at `/etc/wendy-agent/device-key.pem` and reused on re-provisioning.

---

## WendyTelemetryService

`Proto/wendy/agent/services/v1/wendy_agent_v1_telemetry_service.proto`

Streams OTLP telemetry from the device to CLI clients. Data originates from:
- Container stdout/stderr captured by the container log manager.
- OTLP data pushed by containers to the agent's OTLP receivers (gRPC `:4317`, HTTP `:4318`).
- Agent-internal logs (tee'd into the broadcaster at startup).
- Agent process and container CPU/memory metrics (collected every 15 seconds).

| Method | Type | Description |
|---|---|---|
| `StreamLogs` | Server-stream | Streams `ExportLogsServiceRequest` OTLP envelopes. Filterable by `service_name`, `min_severity`, and `app_name`. |
| `StreamMetrics` | Server-stream | Streams `ExportMetricsServiceRequest` OTLP envelopes. Filterable by `service_name`, `metric_name_prefix`, and `app_name`. |
| `StreamTraces` | Server-stream | Streams `ExportTraceServiceRequest` OTLP envelopes. Filterable by `service_name`, `app_name`, and `span_name_prefix`. |

---

## WendyFileSyncService

`Proto/wendy/agent/services/v1/wendy_agent_v1_file_sync_service.proto`

Bidirectional file synchronisation used by the CLI to push source trees to a running container.

| Method | Type | Description |
|---|---|---|
| `SyncFiles` | Bidi-stream | Client sends `FileSyncStart` (with a manifest), then `FileSyncChunk`/`FileSyncCommit` pairs for changed files, optional `FileSyncChmod` for mode-only changes, and optional `FileSyncDelete` for stale files. Agent replies with `FileSyncManifest`, `FileSyncAck` per file, then `FileSyncComplete`. |

---

## OpenTelemetry receiver endpoints

The agent registers the standard OTLP collector services on a dedicated gRPC server (`:4317`):

| Service | Proto package |
|---|---|
| `LogsService` | `opentelemetry.proto.collector.logs.v1` |
| `MetricsService` | `opentelemetry.proto.collector.metrics.v1` |
| `TraceService` | `opentelemetry.proto.collector.trace.v1` |

An HTTP/protobuf receiver on `:4318` accepts the same three signal types at the standard OTLP/HTTP paths (`/v1/logs`, `/v1/metrics`, `/v1/traces`).

---

## Bluetooth L2CAP channel

On provisioned devices the agent additionally exposes a Bluetooth L2CAP server protected by the device mTLS certificate. It accepts `BluetoothCommand` protobuf messages and returns `BluetoothResponse` messages (defined in `wendy_agent_v1_bluetooth.proto`). The commands mirror the gRPC WiFi, Bluetooth, and app management methods, allowing the CLI to control the device over BLE before a network connection is established.
