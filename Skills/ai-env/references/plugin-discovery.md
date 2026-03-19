# Plugin Discovery Reference

This reference covers how to find and recommend plugins in response to a user
request. The goal is to suggest only plugins that are genuinely needed —
avoid recommending extras that don't directly serve the user's stated intent.

---

## When to run discovery

Run plugin discovery when the user:
- Asks to "find plugins" or "what plugins should I add"
- Describes a capability they need and asks how to get it
- Asks to "set up my environment for X"
- Mentions tools or services and you need to check if a plugin covers them

Do NOT run discovery when the user asks to install a specific plugin by name —
just add it to `ai-env.json` directly.

---

## Step 1 — Understand what the user actually needs

Before searching any marketplace, analyse the user's intent:

1. List the concrete tasks the user wants to accomplish
2. Identify the tools or services involved (e.g. "work with Word docs", "query
   our Jira project", "generate PDFs from reports")
3. Separate the tasks into: already covered by skills/MCPs in `ai-env.json`
   vs genuinely missing capability

Write out this analysis before proceeding. Example:

```
User wants to: generate weekly reports as Word docs and send to Slack
Already covered: none
Missing capability: Word document generation, Slack messaging
Services involved: Microsoft Word (.docx), Slack
```

Only search for plugins that address the **missing capabilities**.

---

## Step 2 — Check installed marketplaces

Read the `marketplaces` array from `ai-env.json`. These are the only
sources available to this environment. Do not suggest plugins from marketplaces
that are not declared.

For each declared marketplace:

**Anthropic marketplace** (`https://claude.ai/plugins`):
- Browse or search via the Claude Code UI
- Check the known plugins list below as a starting point

**npm registry** (`https://registry.npmjs.org`):
- This is for MCP servers (npx transport), not plugins
- Skip for plugin discovery

**PyPI** (`https://pypi.org`):
- This is for MCP servers (uvx transport), not plugins
- Skip for plugin discovery

**Docker Hub MCP** (`https://hub.docker.com/mcp`):
- This is for MCP server images, not plugins
- Skip for plugin discovery

**Private marketplace**:
- Ask the user to describe what plugins are available there
- Or ask them to share the marketplace catalogue URL

---

## Step 3 — Match capabilities to plugins

For each missing capability, look for a plugin that covers it. Use the known
plugin table below as reference. Search the marketplace if the capability is
not in the table.

**Known Anthropic marketplace plugins:**

| Plugin ID | Provides | Use when user needs |
|-----------|----------|---------------------|
| `ms-office-suite` | docx, xlsx, pptx, pdf skills | Word, Excel, PowerPoint, PDF generation or editing |

When searching the Anthropic marketplace:
- Check https://claude.ai/plugins (ask the user to browse if you cannot access it)
- Match by the capability description, not by plugin name alone
- One plugin may cover multiple needs — check what each plugin provides before
  recommending separate ones

---

## Step 4 — Filter: keep only what is genuinely needed

Before finalising recommendations, apply this checklist for each candidate
plugin:

**Include the plugin if:**
- It directly provides a capability the user named
- There is no simpler way to get the same capability (e.g. via an MCP server
  alone, without a full plugin)
- The user's workflow will not work without it

**Exclude the plugin if:**
- The capability it provides is already covered by a skill or MCP already in
  `ai-env.json`
- The plugin provides the capability only as a side effect — the user did not
  ask for that capability
- A targeted MCP server would satisfy the need with less overhead than a
  full plugin bundle
- The plugin seems useful in general but does not map to a specific stated need

The goal is the smallest set of plugins that satisfies the stated needs.
Avoid recommending plugins "just in case" or because they look related.

---

## Step 5 — Present findings and confirm

Before adding anything to `ai-env.json`, present the proposed plugins to
the user:

```
Based on your request, here are the plugins I recommend:

✓ ms-office-suite (Anthropic marketplace)
  Provides: docx, xlsx, pptx, pdf skills
  Needed for: generating Word reports and PDF exports

✗ Not recommended: <other plugin>
  Reason: its Slack integration is already covered by the atlassian MCP

Shall I add these to ai-env.json?
```

Wait for the user to confirm before writing to `ai-env.json`.

---

## Step 6 — Add confirmed plugins to config

After confirmation, add each approved plugin to the `plugins` array in
`ai-env.json`. Reference the correct marketplace name:

```json
{
  "name": "ms-office-suite",
  "marketplace": "anthropic",
  "id": "ms-office-suite"
}
```

Then check if the plugin provides any skills needed in `skills` — if so,
switch those skill entries to `source: "plugin"` to avoid duplicate installs.

Run `config-validation.md` checks after editing.

---

## Handling unknown marketplaces

If the user has a private marketplace declared but you cannot access its
catalogue:

1. Ask the user: "What plugins are available in your <marketplace-name>
   marketplace? I need to know what capabilities each one provides."
2. Once the user describes the available plugins, apply the same Steps 3–5
   to select the right ones
3. Never guess plugin IDs for private marketplaces — only use IDs the user
   confirms exist

---

## Anti-patterns to avoid

- **Do not recommend a plugin because its name matches a keyword** — check
  what it actually provides
- **Do not install multiple plugins that provide the same capability** —
  pick the one that best fits, report the others as alternatives
- **Do not add plugins to "future-proof" the environment** — install only
  what is needed now
- **Do not recommend plugins from marketplaces not in `ai-env.json`** —
  the user has not authorised those sources
