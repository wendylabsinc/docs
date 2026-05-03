# `wendy cloud discover`

Lists enrolled compute devices visible to the authenticated Wendy Cloud organization.

## Usage

```sh
wendy cloud discover [flags]
```

## Description

`wendy cloud discover` queries the Wendy Cloud asset service for devices registered in your organization. By default **only online devices** (those with an active broker presence) are returned. Pass `--all` to include devices that are currently offline.

When no devices match the filter, a contextual message is printed:

- **Default (online only):** `No online devices found. Use --all to include offline devices.`
- **With `--all`:** `No enrolled devices found.`

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--all` | `false` | Include offline devices in the results. When omitted only devices with an active broker presence are shown. |
| `--cloud-grpc` | `""` | Cloud gRPC endpoint. Required when multiple auth sessions exist and the correct endpoint is ambiguous. |

## Examples

Show only online devices (default):

```sh
wendy cloud discover
```

Show all enrolled devices, including offline ones:

```sh
wendy cloud discover --all
```

Point at a specific cloud gRPC endpoint:

```sh
wendy cloud discover --cloud-grpc grpc.cloud.wendy.sh:443
```

## How it works

1. Reads the stored mTLS auth configuration (`~/.wendy/config.json`).
2. Constructs a `ListAssetsRequest` scoped to the authenticated organization with `IsComputeDevice = true`.
3. Unless `--all` is passed, sets `OnlineOnly = true` on the request so the server only streams back assets with an active broker presence.
4. Receives assets over a **server-streaming gRPC** call (`AssetService.ListAssets`) and collects them until the stream closes.
5. Prints the resulting device list to stdout.

See also [`wendy cloud tunnel`](./tunnel.md) for opening a tunnel to a discovered device.
