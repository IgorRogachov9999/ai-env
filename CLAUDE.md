# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the **ai-env** Claude Code skill — a declarative environment manager for Claude Code AI projects. It reads `ai-env.json` (desired state), compares it to the actual environment, and applies any differences.

The skill is installed into Claude Code projects under `.claude/skills/ai-env/`. The source in this repo uses `ai-env.json` as the config file name.

## Repository Structure

```
Skills/ai-env/
  SKILL.md              # Main skill prompt — the core logic Claude follows
  schema.json           # JSON Schema for ai-env.json validation and IDE autocomplete
  references/           # Detailed reference docs loaded on demand during apply
    config-validation.md
    config-editing.md
    lockfile.md
    mcp-docker.md / mcp-npm.md / mcp-uvx.md / mcp-config-format.md
    skills-management.md
    plugins-management.md
    plugin-discovery.md
    env-resolution.md
    agents-management.md
.claude-plugin/
  marketplace.json      # Marketplace manifest for distributing this skill as a plugin
```

No build, lint, or test pipeline — this skill is purely markdown and JSON.

## Skill Architecture

The skill manages five resource types processed in strict dependency order:

**marketplaces → plugins → skills → mcpServers → agents**

Later stages depend on earlier ones (e.g. a plugin-sourced skill requires the plugin to be installed first).

### Key design principles

- **Desired state model**: `ai-env.json` declares what should exist; the skill only acts on what differs
- **Idempotent**: safe to run at any time; re-running produces the same result
- **Transport priority for MCP servers**: docker > npx > uvx (Docker preferred for version stability)
- **Skill source priority**: plugin > github > local
- **Lockfile** (`ai-env.lock.json`): tracks last applied state for drift detection; commit this
- **Generated files**: `.mcp.json` and `.env` contain secrets — never commit; must be in `.gitignore`

### Config file (`ai-env.json`) structure

```json
{
  "$schema": ".claude/skills/ai-env/schema.json",
  "marketplaces": [{ "name": "...", "url": "..." }],
  "plugins":      [{ "name": "...", "marketplace": "...", "id": "...", "scope": "project|user|local" }],
  "skills":       [{ "name": "...", "source": "github|plugin|local", "repo": "owner/repo" }],
  "mcpServers":   [{ "name": "...", "transport": "docker|npx|uvx|local", "image|package|command": "..." }],
  "agents":       [{ "name": "...", "path": ".claude/agents/name.md" }]
}
```

### Environment variable resolution

`env` values prefixed with `$` in `mcpServers` are resolved from `.env` at project root. If `.env` is missing or a variable is absent, apply stops and lists every unresolved variable.

## Editing the Skill

When modifying `SKILL.md` or reference files:

- `SKILL.md` is the authoritative prompt — it references specific reference files by path. Keep those paths accurate.
- Reference files are loaded **on demand** (not all upfront). The table at the bottom of `SKILL.md` maps each file to the situation that triggers it.
- `schema.json` must stay in sync with the field definitions described in `references/config-validation.md`.
- The output symbols (`✓ + - ~ ! ↩`) are used consistently across all output sections — preserve them when editing output format documentation.
