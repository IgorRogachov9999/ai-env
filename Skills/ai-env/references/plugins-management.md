# Plugins Management Reference

## What plugins are

Claude Code plugins are installable bundles that extend the environment with
MCPs, skills, and tools in a single package. Installing a plugin can provide
multiple MCP servers and skills simultaneously without individual configuration.

Plugins are distributed through marketplaces and managed by Claude Code itself,
not by files on disk.

## Marketplace: Anthropic

The official Claude Code plugin marketplace:

- **URL:** https://claude.ai/plugins
- **Browse:** Available from the Claude Code settings or the Cowork sidebar
- **Auth:** Requires a Claude account (same account used for Claude Code)

## Checking if a plugin is installed

Claude Code tracks installed plugins internally. To check programmatically,
look for the plugin's bundled skills or MCPs being available:

```bash
# If the plugin installs a skill, check for it
test -d .claude/skills/<plugin-skill-name> && echo "plugin likely installed"
```

For a definitive check, ask the user to verify in Claude Code settings →
Plugins, or check the Cowork sidebar. There is no CLI command to list installed
plugins.

## Installing a plugin

Plugins cannot be installed programmatically by `ai-env`. Installation
requires user interaction in the Claude Code UI.

When a required plugin is not installed:
1. Report it as a `!` item in apply output
2. Tell the user which marketplace and plugin ID to install
3. Provide the direct URL if available
4. After the user installs, re-run apply

Example output:
```
! ms-office-suite   not installed
  Install from: https://claude.ai/plugins
  Plugin ID: ms-office-suite
  Then re-run ai-env to apply
```

## Plugin-provided MCPs

Plugins often bundle MCP servers. When a plugin is installed, its MCPs become
available automatically — they do not need entries in `.mcp.json`.

During apply, if an MCP server name appears in both the plugin-provided
list and `ai-env.json`, the explicitly declared entry in `ai-env.json`
takes priority. The plugin-provided entry is silently suppressed.

If two plugins provide the same MCP server name, the first plugin in the
`plugins` array of `ai-env.json` wins. Report the conflict as `~` drift.

## Plugin-provided skills

Skills provided by plugins are available to Claude Code automatically.
They do not live in `.claude/skills/` and do not appear in git.

Verify a plugin skill is active by attempting to use it — if the plugin is
installed, the skill will be available. If not, Claude Code will report the
skill as unknown.

## Known plugins and their contents

| Plugin ID | Marketplace | Provides skills | Provides MCPs |
|-----------|-------------|-----------------|---------------|
| `ms-office-suite` | anthropic | `docx`, `xlsx`, `pptx`, `pdf` | — |

This table reflects known plugins at the time of writing. For the current list,
check https://claude.ai/plugins.

## Private / custom marketplaces

Some organisations host private plugin marketplaces. These are configured in
Claude Code settings and behave the same as the Anthropic marketplace during apply.

In `ai-env.json`, declare them in `marketplaces` with their URL:

```json
{
  "name": "my-org",
  "url": "https://plugins.internal.mycompany.com"
}
```

Plugins from private marketplaces follow the same install flow as Anthropic
plugins — user interaction required, `ai-env` only reports missing ones.
