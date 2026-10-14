# Configuration snippets

All of them point to the same remote server; there is nothing to install.

## Claude Code

```
claude mcp add --transport http iescrow https://api.iescrow.io/mcp
```

## Generic `mcp.json` (Cursor, Windsurf, VS Code and other hosts that read it)

```json
{
  "mcpServers": {
    "iescrow": {
      "url": "https://api.iescrow.io/mcp"
    }
  }
}
```

Hosts that use the `type` field:

```json
{
  "mcpServers": {
    "iescrow": {
      "type": "streamable-http",
      "url": "https://api.iescrow.io/mcp"
    }
  }
}
```

## MCP Inspector (to explore the tools)

```
npx @modelcontextprotocol/inspector
```

Transport *Streamable HTTP*, URL `https://api.iescrow.io/mcp`, then *Open Auth Settings* → *Quick OAuth Flow*. The inspector registers itself as a client and sends you to the iEscrow consent screen.
