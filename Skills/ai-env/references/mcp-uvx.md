# MCP Server Management — UVX Transport

## What is uvx

`uvx` is the tool runner from [uv](https://github.com/astral-sh/uv) — a fast
Python package manager. It runs Python-based CLI tools without requiring a
virtual environment or `pip install`, similar to `npx` for Node.

## Prerequisites

Check uv and uvx are available:

```bash
uvx --version
```

If uvx is not available, install uv first:

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

After install, reload shell: `source $HOME/.local/bin/env` (macOS/Linux)

If uv cannot be installed automatically, report it as a `!` blocker for all
uvx-based servers and continue with the remaining resources.

## .mcp.json format for uvx transport

```json
{
  "server-name": {
    "command": "uvx",
    "args": ["package-name"],
    "env": {
      "KEY": "value"
    }
  }
}
```

Note: unlike npx, uvx does not need a `-y` flag. It installs and runs
the package non-interactively by default.

## Pinning a specific version

```json
"args": ["package-name==1.2.3"]
```

## Finding uvx-based MCP servers

Browse available servers at:
- https://pypi.org — search `mcp-server` or `mcp`
- https://hub.docker.com/mcp (prefer Docker image if available — see transport priority)

Check the package README for the required `env` keys before adding to `.mcp.json`.

## Verifying a package before adding

```bash
uvx package-name --version 2>/dev/null || echo "not found"
```

Or check the PyPI registry directly:

```bash
pip index versions package-name 2>/dev/null | head -1
```

## Troubleshooting

If a uvx-based server fails to start:

1. Test manually: `uvx package-name`
2. Check the package on PyPI: `https://pypi.org/project/package-name/`
3. Verify uv is up to date: `uv self update`
4. Check Python version: `python --version` (Python 3.8+ required for most servers)
5. Try explicit Python version: `uvx --python 3.11 package-name`
