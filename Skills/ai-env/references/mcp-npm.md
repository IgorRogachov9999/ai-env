# MCP Server Management — NPX / NPM Transport

## Prerequisites

Check Node.js and npx are available:

```bash
node --version
npx --version
```

If npx is not available, report it as a `!` blocker for all npx-based servers.
Node.js 18+ is recommended. Most MCP servers require at least Node.js 16.

## .mcp.json format for npx transport

```json
{
  "server-name": {
    "command": "npx",
    "args": ["-y", "@scope/package-name"],
    "env": {
      "KEY": "value"
    }
  }
}
```

Key flags:
- `-y` — skip confirmation prompt (required for non-interactive use)
- Use the full scoped package name, e.g. `@modelcontextprotocol/server-github`
- `env` is passed as a JSON object, not as CLI flags

## Verifying a package exists before adding to .mcp.json

```bash
npm view @scope/package-name version 2>/dev/null || echo "not found"
```

Claude Code handles the actual install on first invocation via npx's on-demand
install. No explicit `npm install` step is needed.

## Pinning a specific version

To use a specific version instead of latest, include the version in the package name:

```json
"args": ["-y", "@scope/package-name@1.2.3"]
```

Omitting the version always pulls the latest published version.

## Finding npx-based MCP servers

Browse available servers at:
- https://github.com/modelcontextprotocol/servers (official reference servers)
- https://hub.docker.com/mcp (prefer Docker image if available — see transport priority)
- npm registry: `npm search mcp-server`

Check the package README for the required `env` keys before adding to `.mcp.json`.

## Package registry

All npx-based servers resolve from the npm public registry by default:
`https://registry.npmjs.org`

No additional registry configuration is needed unless the project uses a
private registry.

## Troubleshooting

If an npx-based server fails to start:

1. Test manually: `npx -y package-name`
2. Check the package exists: `npm view package-name`
3. Clear npx cache if stale: `npx clear-npx-cache`
4. Check Node version compatibility: `node --version`
