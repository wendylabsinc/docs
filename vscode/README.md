# AI Coding Agent Setup

Use `wendy mcp setup` to connect any supported AI coding assistant to your Wendy devices. One command detects installed tools and writes the `wendy mcp serve` invocation into each one's configuration.

```sh
wendy mcp setup
```

## Supported tools

| Tool | Config written |
|------|---------------|
| Claude Code | `~/.claude.json` |
| Claude Desktop | `claude_desktop_config.json` (platform default) |
| Cursor | `~/.cursor/mcp.json` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| Codex | `~/.codex/config.toml` |

Detection uses the presence of the tool's config directory or its binary on `PATH`. Only detected tools are configured. The binary path is recorded without resolving symlinks, so the configuration remains valid across `wendy` upgrades.

`wendy mcp setup` also installs bundled Wendy skills into supported tools (Claude Code, Codex, OpenCode). Skills are automatically refreshed when the CLI is upgraded — you don't need to re-run setup after updating `wendy`.

Sample output:

```
✓ Claude Code: configured at /Users/you/.claude.json
✓ Claude Desktop: configured at /Users/you/Library/Application Support/Claude/claude_desktop_config.json
```

## What it connects you to

`wendy mcp setup` wires in `wendy mcp serve`, which exposes:

- Device management (connect, list, info, set default)
- Container lifecycle (start, stop, delete, attach, stats)
- Telemetry (logs, metrics, traces)
- WiFi, Bluetooth, and hardware tools
- OS updates and file sync
- Cloud enrollment and tunnel access
- MCP tools from running app containers (proxied automatically)

See the [MCP reference](../wendy-agent/mcp.md) for the full tool inventory and advanced configuration.

## Manual setup

If your tool is not detected automatically, add the following to its MCP server configuration:

```json
{
  "mcpServers": {
    "wendy": {
      "command": "/path/to/wendy",
      "args": ["mcp", "serve"]
    }
  }
}
```

For Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.wendy]
command = "wendy"
args = ["mcp", "serve"]
```

## Starting a session

Once configured, start your AI coding tool and call `wendy_status` first — it returns the current connection state and tells you what to do next:

```
> wendy_status

{
  "connected": false,
  "suggested_next_step": "not connected — call device_list to see available devices, then device_connect"
}
```

From there, `device_list` discovers devices, `device_connect` establishes a connection, and all other tools become available.
