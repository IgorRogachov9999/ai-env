# ai-env

A Claude Code skill that manages your AI development environment using a declarative desired-state model. Define what you need in `ai-env.json` — marketplaces, plugins, skills, MCP servers, and agents — and `ai-env` applies the config to match.

## How it works

`ai-env.json` is the single source of truth. The skill reads it, compares it to the current environment, and acts only on what differs. Running it multiple times is safe — it only changes what needs changing.

**Resource types, processed in order:**

| Type | What it does |
|------|-------------|
| `marketplaces` | Plugin registries to connect to |
| `plugins` | Plugins installed from marketplaces |
| `skills` | Claude Code skills (GitHub clone, plugin, or local) |
| `mcpServers` | MCP servers written to `.mcp.json` |
| `agents` | Subagent `.md` files validated in `.claude/agents/` |

## Installation

### Via marketplace plugin

Add to your `ai-env.json`:

```json
{
  "marketplaces": [
    { "name": "ai-env-marketplace", "url": "https://github.com/IgorRogachov9999/ai-env" }
  ],
  "plugins": [
    { "name": "ai-env", "marketplace": "ai-env-marketplace", "id": "ai-env" }
  ]
}
```

### Manual install

```bash
git clone --depth 1 https://github.com/IgorRogachov9999/ai-env .claude/skills/ai-env
```

## Usage

Create `ai-env.json` in your project root:

```json
{
  "$schema": ".claude/skills/ai-env/schema.json",
  "marketplaces": [
    { "name": "anthropic", "url": "https://claude.ai/plugins" }
  ],
  "plugins": [
    { "name": "ms-office-suite", "marketplace": "anthropic", "id": "ms-office-suite" }
  ],
  "skills": [
    { "name": "my-skill", "source": "github", "repo": "owner/repo" }
  ],
  "mcpServers": [
    { "name": "my-server", "transport": "docker", "image": "myorg/my-mcp:latest" }
  ],
  "agents": [
    { "name": "backend-dev", "path": ".claude/agents/backend-dev.md" }
  ]
}
```

Then ask Claude Code to apply:

> "Apply ai-env" / "Set up my AI environment" / "Sync ai-env.json"

### Commands

| Phrase | What happens |
|--------|-------------|
| *(default)* | Apply `ai-env.json` to the environment |
| `plan` / `dry-run` | Preview changes without applying them |
| `status` | Show current environment state |
| `add X` / `remove Y` | Edit `ai-env.json` via natural language |
| `find plugins for X` | Discover and recommend plugins |

## Files

| File | Purpose | Commit? |
|------|---------|---------|
| `ai-env.json` | Desired state config | Yes |
| `ai-env.lock.json` | Last applied versions | Yes |
| `.mcp.json` | Generated MCP config | No |
| `.env` | Secret values for MCP servers | No |

Make sure `.mcp.json` and `.env` are in your `.gitignore`.

## MCP server transport priority

When multiple transports are available, `ai-env` prefers: **docker > npx > uvx**

```json
{ "name": "my-server", "transport": "docker", "image": "myorg/server:1.0.0" }
{ "name": "my-server", "transport": "npx",    "package": "@myorg/server" }
{ "name": "my-server", "transport": "uvx",    "package": "myorg-server" }
```

## Schema

Add `"$schema": ".claude/skills/ai-env/schema.json"` to your `ai-env.json` for IDE validation and autocomplete.

## License

[MIT](LICENSE)
