# Cloud Tunnel

`wendy cloud tunnel` opens a secure tunnel to a cloud-managed device, forwarding a local port through the Wendy Cloud broker to the device.

## Usage

```sh
wendy cloud tunnel [flags]
```

If no device is specified, an interactive picker lists all enrolled cloud devices and lets you select one.

## Device picker asset limit

To prevent unbounded memory growth from a misbehaving backend, the device picker fetches **at most 10 000 devices** from the cloud. If the backend returns more than 10 000 assets, the command exits with an error:

```
cloud returned more than 10000 devices
```

## Flags

| Flag | Description |
|------|-------------|
| `--device-id` | ID of the target cloud device (skips the interactive picker). |
| `--port` | Local port to forward. |
| `--broker` | Cloud broker address (default port `50052`). |

## How it works

1. Authenticates with Wendy Cloud using the stored mTLS certificate (`~/.wendy/config.json`).
2. Fetches the list of enrolled devices from the cloud (capped at 10 000).
3. Prompts the user to select a device if `--device-id` is not provided.
4. Establishes a bidirectional gRPC stream to the cloud broker.
5. Forwards TCP traffic from the local port through the broker to the device.

## Related

- [`wendy cloud discover`](./discover.md) — list all enrolled cloud devices
- [Cloud connectivity](./connectivity.md)
- [Cloud requirements](./requirements.md)
