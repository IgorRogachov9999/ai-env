# MCP Server Management — Docker Transport

## Prerequisites

Check Docker is available before attempting any Docker-based MCP operation:

```bash
docker --version
```

If Docker is not installed, report it as a `!` blocker for all Docker-based
servers and continue with the remaining resources.

## .mcp.json format for Docker transport

```json
{
  "server-name": {
    "command": "docker",
    "args": [
      "run", "--rm", "-i",
      "--env", "KEY=value",
      "--env", "KEY2=value2",
      "image:tag"
    ]
  }
}
```

Key flags:
- `--rm` — remove the container when it exits (MCP servers are short-lived)
- `-i` — keep stdin open (required for MCP stdio transport)
- Do not use `-d` (detached) — MCP servers communicate over stdio, not sockets
- Do not use `-t` (tty) — breaks stdio communication

## Checking whether an image is available locally

```bash
docker image inspect image:tag 2>/dev/null | grep -q "Id" && echo "present" || echo "missing"
```

Claude Code pulls Docker images on first use via the `docker run` command,
so no explicit `docker pull` is required. However, you can pre-pull to verify
the image exists and avoid first-run latency:

```bash
docker pull image:tag
```

## Passing environment variables

Prefer `--env KEY=value` per variable over `--env-file` for MCP servers,
because the values come from `.env` and are resolved at config-generation time.

If many variables are needed, generate them as a list of `--env` pairs:

```bash
# Pattern for multiple env vars
--env VAR1=value1 --env VAR2=value2 --env VAR3=value3
```

## Finding Docker-based MCP servers

Browse available images at:
- https://hub.docker.com/mcp — official Docker MCP catalog (`mcp/<server-name>`)
- GitHub Container Registry (ghcr.io) — many open-source MCP servers publish images here

Check the image README for required `--env` variables before adding to `.mcp.json`.

## Troubleshooting

If a Docker-based MCP server fails to start:

1. Verify image exists: `docker image ls | grep <image-name>`
2. Test run manually: `docker run --rm -i --env KEY=value image:tag`
3. Check Docker daemon is running: `docker info`
4. Inspect available ports if the server requires network access
