# Cloud Discover

`wendy cloud discover` lists devices enrolled in Wendy Cloud.

## Usage

```sh
wendy cloud discover [flags]
```

## Output

Results are **streamed** to the terminal as each device arrives from the backend — rows appear immediately rather than waiting for the full response to buffer. The header is printed before the first result row.

```
ID        Name
--------  ----
1042      my-jetson
1099      pi-lab-01
```

If no devices are found, a descriptive message is printed instead:

- Without `--all`: `No online devices found. Use --all to include offline devices.`
- With `--all`: `No enrolled devices found.`

## Asset limit

To prevent unbounded memory growth from a misbehaving backend, `cloud discover` caps results at **10 000 devices**. If the backend streams more than 10 000 assets, the command exits with an error:

```
cloud returned more than 10000 devices
```

## Flags

| Flag | Description |
|------|-------------|
| `--all` | Include offline (not currently connected) devices. By default only online devices are shown. |

## Related

- [`wendy cloud tunnel`](./tunnel.md) — open a tunnel to a cloud device
- [Cloud connectivity](./connectivity.md)
- [Cloud requirements](./requirements.md)
