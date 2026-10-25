# MCP Servers in Claude Desktop (Weather Example)

This note summarizes how MCP servers work, based on the `weather` exercise in this repo.

## 1) What an MCP server is

An MCP server is a local process that exposes tools/resources to an MCP client (Claude Desktop).

In this project, the MCP server logic is in:

- `weather.py` (repo root)

It defines:

- `mcp = MCPServer("weather")`
- tools via `@mcp.tool()`:
  - `get_alerts(state: str)`
  - `get_forecast(latitude: float, longitude: float)`
- server startup via:
  - `mcp.run(transport="stdio")`

## 2) How Claude Desktop connects (stdio transport)

Claude Desktop starts the server command you configure, then communicates over process pipes:

- Claude writes MCP messages to **server stdin**
- Server writes responses to **server stdout**

Messages are JSON-RPC style JSON objects (one message per line in this SDK’s stdio transport).

### Message shapes

- Request:
```json
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}
```

- Response:
```json
{"jsonrpc":"2.0","id":1,"result":{...}}
```

- Notification:
```json
{"jsonrpc":"2.0","method":"notifications/initialized"}
```

## 3) Handshake + tool discovery flow

Typical startup flow:

1. `initialize` (client -> server)
2. `initialize` result (server -> client)
3. `notifications/initialized` (client -> server)
4. `tools/list` (client -> server)
5. `tools/list` result (server -> client)

`tools/list` returns each tool’s:

- name
- description
- input schema

The schema is generated from your function signature and metadata from `@mcp.tool()`.

## 4) Tool calling flow

When Claude chooses a tool:

1. Sends `tools/call` with `name` + `arguments`
2. Server validates args against tool schema
3. Your Python function runs
4. Server returns structured tool result content
5. Claude uses that output to answer the user

## 5) Weather example transcript (condensed)

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"claude-desktop","version":"x.y.z"}}}
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-03-26","capabilities":{"tools":{}},"serverInfo":{"name":"weather","version":"0.1.0"}}}
{"jsonrpc":"2.0","method":"notifications/initialized"}
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
{"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"get_alerts",...},{"name":"get_forecast",...}]}}
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_forecast","arguments":{"latitude":37.7749,"longitude":-122.4194}}}
{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"...forecast text..."}]}}
```

## 6) Important repo-specific gotcha

Current entrypoint setup in `pyproject.toml` maps:

- `weather = "weather:main"`

But `src/weather/__init__.py` currently just prints:

- `Hello from weather!`

So this console script does **not** start MCP right now. The MCP server is in root `weather.py`.

## 7) Claude Desktop config example for this repo

`~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "weather": {
      "command": "/Users/ashbas/weather/.venv/bin/python",
      "args": ["/Users/ashbas/weather/weather.py"]
    }
  }
}
```

## 8) Practical tips

- Keep stdout clean while serving MCP (avoid random `print()` noise to stdout).
- Use logging/stderr for debug messages.
- Make tool names/descriptions clear; Claude uses them for tool selection.
- Add type hints + docstrings so tool schemas are accurate.
- Handle API failures gracefully and return useful error text.

---

Created from the MCP weather walkthrough in this repo.
