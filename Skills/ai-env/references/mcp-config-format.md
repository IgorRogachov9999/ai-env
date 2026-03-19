# .mcp.json — Configuration Format Reference

## File location and scope

Claude Code reads MCP configuration from two locations, in priority order:

1. **Project-level** — `.mcp.json` in the project root directory.
   Applies only when working in that project. **Do not commit** — it contains
   resolved secret values. Use `ai-env.json` as the committed file instead.

2. **User-level** — `~/.claude/mcp.json` (macOS/Linux) or
   `%APPDATA%\Claude\mcp.json` (Windows).
   Applies globally across all projects.

`ai-env` manages the **project-level** `.mcp.json`.

## Full schema

```jsonc
{
  "mcpServers": {
    "<server-name>": {
      "command": "<executable>",
      "args": ["<arg1>", "<arg2>"],
      "env": {
        "<KEY>": "<value>"
      },
      "cwd": "<optional-working-directory>"
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | string | yes | Executable to run (`npx`, `uvx`, `docker`, `node`, `python`, etc.) |
| `args` | array | yes | Command arguments. For npx: `["-y", "package"]`. For uvx: `["package"]`. For docker: see docker reference. |
| `env` | object | no | Environment variables passed to the server process. Values must be resolved strings — no `$VARIABLE` placeholders. |
| `cwd` | string | no | Working directory for the server process. Defaults to project root. |

## Transport patterns

### docker (preferred)
```json
{
  "my-server": {
    "command": "docker",
    "args": [
      "run", "--rm", "-i",
      "--env", "API_KEY=...",
      "--env", "BASE_URL=https://example.com",
      "mcp/my-server:latest"
    ]
  }
}
```

### npx (Node.js packages)
```json
{
  "my-server": {
    "command": "npx",
    "args": ["-y", "@scope/mcp-server-name"],
    "env": { "API_KEY": "..." }
  }
}
```

### uvx (Python packages)
```json
{
  "my-server": {
    "command": "uvx",
    "args": ["mcp-package-name"],
    "env": {
      "API_KEY": "...",
      "REGION": "..."
    }
  }
}
```

### local binary or script
```json
{
  "my-server": {
    "command": "node",
    "args": ["/path/to/server.js"],
    "env": { "PORT": "3000" }
  }
}
```

## Server naming rules

- Use lowercase kebab-case names: `github`, `aws`, `zephyr-scale`
- Names must be unique within the file
- The name is how Claude Code identifies and references the server
- Changing a name is equivalent to removing the old server and adding a new one

## What Claude Code does with .mcp.json

Claude Code reads `.mcp.json` at session start and makes all declared servers
available as tools. Servers are started on demand (lazily) when a tool from
that server is first called. They are not started eagerly at session start.

If a server fails to start, Claude Code logs the error and continues —
the server's tools will show as unavailable but other servers remain functional.

## Secrets and .gitignore

`.mcp.json` at project level may contain resolved secret values.
Ensure `.gitignore` includes:

```
.mcp.json
.env
```

If the project should share MCP server _declarations_ (without secrets),
use `ai-env.json` as the committed file and generate `.mcp.json`
locally by resolving from `.env`.
