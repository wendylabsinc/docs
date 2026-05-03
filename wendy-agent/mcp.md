# MCP (Model Context Protocol)

The wendy agent exposes device tools to AI assistants via the [Model Context Protocol (MCP)](https://modelcontextprotocol.io). Apps that declare an `mcp` entitlement in `wendy.json` are automatically discoverable through `wendy mcp serve`.

## How it works

```
AI Assistant
    │  MCP (stdio)
    ▼
wendy mcp serve
    │  StreamMCP gRPC (bidirectional streaming)
    ▼
wendy-agent (on device)
    │  TCP
    ▼
container MCP server (127.0.0.1:<port>)
```

The wendy agent's `StreamMCP` RPC proxies raw bytes between the gRPC stream and the container's TCP port. Before forwarding, the agent verifies:

- The container has an `mcp` entitlement (non-zero `sh.wendy/mcp.port` label).
- The container is in the `RUNNING` state.

## Cloud asset listing (`list_cloud_assets`)

The MCP tool `list_cloud_assets` enumerates devices enrolled in Wendy Cloud. To prevent unbounded memory growth from a misbehaving backend, results are capped at **10 000 devices**. If the backend returns more than 10 000 assets, the tool returns an error:

```
cloud returned more than 10000 devices
```

## Enabling MCP for an app

Add the `mcp` entitlement to `wendy.json`:

```json
{
  "entitlements": [
    { "type": "network", "mode": "host" },
    { "type": "mcp", "port": 3000 }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `port` | integer | TCP port on which the container's MCP server listens (required). |

The `network` (host mode) entitlement is required so the agent can reach the container's MCP port over loopback.

The container must serve the [MCP Streamable HTTP](https://modelcontextprotocol.io/specification) transport on `0.0.0.0:<port>`.

## Running the MCP server

```sh
wendy mcp serve
```

Tools from all running containers with an `mcp` entitlement appear prefixed with the app ID (e.g. `com.wendylabs.examples.mcp-example/ping`).

## Reference implementation

See [Examples/MCPExample](../Examples/MCPExample/README.md) for a complete Python reference implementation using FastMCP and uvicorn.

## gRPC API

| RPC | Description |
|-----|-------------|
| `StreamMCP` | Bidirectional streaming RPC that proxies raw MCP bytes between the caller and the container's TCP port. |

## Related

- [wendy.json — `mcp` entitlement](../apps/wendy.json.md#mcp)
- [MCPExample](../Examples/MCPExample/README.md)
- [`wendy mcp serve`](../clients/wendy-cli/commands/)
