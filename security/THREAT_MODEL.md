# Threat Model

## Asset result limits

To prevent unbounded memory growth from a misbehaving or compromised cloud backend, all code paths that consume streaming cloud device lists enforce a hard cap of **10 000 assets**:

| Code path | Behaviour when limit exceeded |
|-----------|-------------------------------|
| `wendy cloud discover` | Command exits with error: `cloud returned more than 10000 devices` |
| `wendy cloud tunnel` (device picker) | Command exits with error: `cloud returned more than 10000 devices` |
| MCP `list_cloud_assets` | Tool returns error: `cloud returned more than 10000 devices` |

This cap is enforced on the client side regardless of server behaviour.

## Streaming output (`cloud discover`)

`wendy cloud discover` prints rows to stdout **as they arrive** from the streaming gRPC response rather than buffering the full result set. This means:

- Memory usage is O(1) with respect to the number of results (header and current row only).
- Output begins immediately; the user sees devices without waiting for the stream to complete.
- The 10 000-asset cap is still enforced: if the backend sends more than 10 000 assets, the stream is terminated with an error before additional assets are processed.
