# wendy.json

`wendy.json` is the configuration file for a Wendy app. It lives at the root of your project and describes the app's identity, target platform, required capabilities, and runtime behaviour.

## Example

```json
{
  "$schema": "https://wendy.sh/schemas/wendy.json",
  "appId": "my-app",
  "version": "1.0.0",
  "language": "swift",
  "entitlements": [
    { "type": "network" },
    { "type": "gpu" }
  ]
}
```

## Fields

### `appId` *(required)*

Unique identifier for the app.

```json
{ "appId": "my-app" }
```

### `version`

Version string for the app, e.g. `"1.0.0"`.

### `platform`

Target platform. One of:

| Value | Description |
|-------|-------------|
| `wendyos` | Linux edge device running WendyOS |
| `wendy-lite` | ESP32 WASM target |

Omit to target the default platform.

### `language`

Project language, e.g. `"swift"` or `"python"`. Used by the CLI to select the appropriate build toolchain.

### `debug`

Set to `true` to enable debug mode (default `false`). Injects debug tooling into the container via the `WENDY_DEBUG` build arg.

### `entitlements`

Array of capabilities the app requires. See [Entitlements](#entitlements-1) below.

### `readiness`

Configures how the CLI determines when the app is ready after starting.

```json
{
  "readiness": {
    "tcpSocket": { "port": 8080 },
    "timeoutSeconds": 30
  }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tcpSocket.port` | integer (1–65535) | — | TCP port to probe |
| `timeoutSeconds` | integer | `30` | How long to wait before giving up |

### `hooks`

Lifecycle commands to run at specific points during the app lifecycle.

```json
{
  "hooks": {
    "postStart": {
      "cli": "open http://localhost:8080",
      "agent": "/app/post-start.sh"
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `hooks.postStart.cli` | Command run on the developer's machine after the app starts |
| `hooks.postStart.agent` | Command run on the device after the app starts |

### `python`

Python-specific settings.

| Field | Description |
|-------|-------------|
| `python.sourceRoot` | Path to the Python source root directory |

### `$schema`

Optional URI pointing to the JSON Schema for editor autocompletion and validation. Set to `"https://wendy.sh/schemas/wendy.json"`.

---

## Entitlements

Entitlements grant the app access to hardware and system capabilities. Any capability not listed is unavailable to the app. They are [code signed](../wendy-agent/oci/codesigning.md), preventing privilege escalation.

Use `wendy project entitlements add` / `remove` to manage them, or edit `wendy.json` directly.

### `network`

IP networking access.

```json
{ "type": "network" }
{ "type": "network", "mode": "host" }
```

| `mode` | Description |
|--------|-------------|
| *(omitted)* | Default isolated network |
| `"host"` | Shares the host network stack |
| `"none"` | Networking fully disabled |

### `gpu`

GPU access for AI inference or general-purpose compute.

```json
{ "type": "gpu" }
```

### `camera`

Camera / V4L2 device access.

```json
{ "type": "camera" }
{ "type": "camera", "allowlist": ["/dev/video0"] }
```

| Field | Description |
|-------|-------------|
| `allowlist` | Restrict access to specific device paths. Omit to allow all cameras. |

### `audio`

Microphone and speaker access.

```json
{ "type": "audio" }
```

### `bluetooth`

Bluetooth access.

```json
{ "type": "bluetooth" }
```

### `persist`

Persistent storage that survives container restarts.

```json
{
  "type": "persist",
  "name": "my-app",
  "path": "/data"
}
```

| Field | Description |
|-------|-------------|
| `name` | Shared namespace (app ID). Apps with the same name can share storage. |
| `path` | Mount path inside the container, e.g. `"/data"`. |

### `usb`

USB device access.

```json
{ "type": "usb" }
```

### `i2c`

I2C bus access.

```json
{ "type": "i2c", "device": "/dev/i2c-1" }
```

| Field | Description |
|-------|-------------|
| `device` | I2C device path (required). |

### `gpio`

GPIO pin access.

```json
{ "type": "gpio" }
{ "type": "gpio", "pins": [17, 27] }
```

| Field | Description |
|-------|-------------|
| `pins` | Pin numbers to expose. Omit to grant access to all GPIO chips. |

### `spi`

SPI device access.

```json
{ "type": "spi" }
```

### `input`

HID input device access (barcode scanners, keyboards, etc.).

```json
{ "type": "input" }
```

---

> **Deprecated:** `{ "type": "video" }` — use `camera` instead.
