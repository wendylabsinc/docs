# Cloud Tunnel

The cloud tunnel lets you connect to enrolled devices without a direct network path. The Wendy Cloud broker relays TCP bytes between your machine and the device's local agent port, authenticated over mTLS.

## Architecture

```
CLI  ──gRPC ClientTunnel stream──►  Cloud Broker  ──TCP──►  Agent (device:50051)
```

Three parties are involved:

1. **CLI** — opens a `ClientTunnel` gRPC stream to the broker and sends a `ClientTunnelOpen` message identifying the target asset and port.
2. **Cloud Broker** — accepts the stream, verifies your identity, and asks the agent on the target device to dial `localhost:<port>`. It then relays bytes bidirectionally.
3. **Agent** — dials the requested local port and forwards bytes through the broker back to the CLI.

The CLI wraps the gRPC stream in a `net.Conn` via `net.Pipe()` and uses it as a gRPC transport (`grpc.WithContextDialer`). The agent-facing gRPC connection has no knowledge that it is tunnelled — it behaves identically to a direct connection.

## Prerequisites

- Your device must be enrolled: `wendy device enroll` or `wendy cloud enroll-device`
- You must be logged in: `wendy auth login`
- The device must be online and connected to Wendy Cloud

## Commands

### `wendy cloud run`

Runs your application on a cloud-enrolled device without a direct network path. Equivalent to `wendy run` but the agent connection goes through the cloud tunnel.

```
wendy cloud run [flags]
```

**Flags** (same as `wendy run`, plus):

| Flag | Default | Description |
|---|---|---|
| `--device <name>` | (picker) | Device name — skips interactive picker |
| `--cloud-grpc <host:port>` | (auth session) | Cloud gRPC endpoint — required when multiple auth sessions exist |
| `--broker-url <host:port>` | `$WENDY_BROKER_URL` or derived from cloud host | Tunnel broker host:port |

If `--device` is omitted and the org has more than one enrolled device, an interactive picker appears. If the org has exactly one device, it is selected automatically.

**Example:**

```bash
wendy cloud run --device my-device
```

### `wendy cloud tunnel`

Forwards a local TCP port to a port on a cloud-enrolled device. Each accepted connection opens a new broker stream.

```
wendy cloud tunnel <local-port>:<remote-port> [flags]
wendy cloud tunnel <port>  # same port on both sides
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--device <name>` | (picker) | Device name — skips interactive picker |
| `--cloud-grpc <host:port>` | (auth session) | Cloud gRPC endpoint |
| `--broker-url <host:port>` | `$WENDY_BROKER_URL` or derived from cloud host | Tunnel broker host:port |

**Examples:**

```bash
# Forward localhost:8080 to port 8080 on the device
wendy cloud tunnel 8080 --device my-device

# Forward localhost:5432 to the device's Postgres port
wendy cloud tunnel 5432:5432 --device my-device

# Forward a different local port
wendy cloud tunnel 15432:5432 --device my-device
```

Press `Ctrl+C` to stop the tunnel listener.

## Broker Endpoint

The broker runs at `<cloud-host>:50053` by default. You can override it with `--broker-url` or the `WENDY_BROKER_URL` environment variable:

```bash
export WENDY_BROKER_URL=broker.example.com:50053
wendy cloud run
```

Ports other than `:443` are treated as local dev brokers and use unencrypted transport. The `:443` port uses mTLS with your enrolled client certificate.

## Authentication

The CLI authenticates to the broker using:

1. **mTLS client certificate** — present when the device is enrolled. The certificate encodes your organisation ID and user identity.
2. **API key** (optional) — if your auth session includes an API key (`apiKey` in `~/.wendy/config.json`), it is sent as a `Bearer` token in the request metadata alongside the certificate.

## Multiple Auth Sessions

If you have auth sessions for multiple cloud environments (e.g. production and staging), use `--cloud-grpc` to select which session to use:

```bash
wendy cloud run --cloud-grpc cloud.staging.example.com:443 --device my-device
```

## Troubleshooting

**"no enrolled devices found for this org"**  
Enroll a device first: `wendy device enroll` or `wendy cloud enroll-device --name my-device`

**"auth entry has no certificates; re-run 'wendy auth login'"**  
Your auth session is missing the client certificate. Log in again: `wendy auth login`

**"connecting to broker at … failed"**  
Check that the broker is reachable from your network and that the port is correct. The default broker port is `50053`.

**Multiple devices, no picker**  
Use `--device <name>` to specify the target directly and skip the interactive picker.
