# Config Editing Reference

This reference covers how to update `ai-env.json` in response to user
instructions in chat. The goal is for the user to express their intent
naturally ("add a Slack MCP", "fix the broken repo link", "remove the
terraform skill") and for Claude to translate that into a safe, validated
edit of the config file.

---

## Edit workflow

Always follow this sequence:

```
1. Read ai-env.json
2. Parse and understand current state
3. Plan the change (explain to user what will be modified)
4. Apply the edit
5. Validate the result (run config-validation.md checks)
6. Report outcome
```

Never write to `ai-env.json` without first reading it. Never skip
validation after editing.

---

## Reading the current config

```bash
cat ai-env.json
```

Or use the Read tool. Load the full file — do not truncate it.

---

## Applying edits safely

Use the Edit tool for targeted changes. Prefer replacing the smallest
possible section to reduce the risk of corrupting surrounding content.

For large structural changes (e.g. reordering arrays, reformatting),
rewrite the full file using the Write tool with the complete updated content.

After any write, verify JSON syntax:

```bash
python3 -c "import json,sys; json.load(open('ai-env.json'))" \
  && echo "OK" || echo "SYNTAX ERROR"
```

If a syntax error is introduced, restore the previous version and report
the failure. Do not leave a broken `ai-env.json`.

---

## Common edit patterns

### Adding a new MCP server

When the user says something like "add an MCP for X" or "I want to connect
Claude to X":

1. Ask (or infer from context) the transport type, package/image, and required env vars
2. Apply Docker transport priority: if a Docker image exists for the server, prefer it
3. Add the entry to the `mcpServers` array:

```json
{
  "name": "server-name",
  "transport": "docker",
  "image": "mcp/server-name:latest",
  "env": {
    "API_KEY": "$SERVER_API_KEY"
  }
}
```

4. If new env vars are needed, add them to `.env.example`:

```bash
echo "SERVER_API_KEY=" >> .env.example
```

And remind the user to set the value in `.env`.

---

### Adding a new skill

When the user says "add skill X" or "I want Claude to know how to do Y":

1. Identify the skill name and source
2. If the skill comes from a known plugin, prefer `source: "plugin"`
3. If it's a GitHub repo, use `source: "github"`
4. Add to the `skills` array:

```json
{
  "name": "skill-name",
  "source": "github",
  "repo": "owner/repo",
  "installPath": ".claude/skills/skill-name"
}
```

If the `repo` URL is uncertain, say so and ask the user to confirm before
writing.

---

### Fixing a broken URL or package name

When validation reports a `!` blocker for a bad repo, image, or package:

1. Search for the correct URL or package name (web search, npm search, PyPI)
2. Show the user the proposed fix:
   `! skill infra-tool: repo acme-org/infra-mcp not found`
   `→ Found: acme-org/infra-tool-mcp — should I update the repo field?`
3. After confirmation, apply the edit

Do not auto-fix URLs silently. Always show the proposed correction first.

---

### Removing an MCP server, skill, or plugin

When the user says "remove X" or "I no longer need Y":

1. Find the entry in the config
2. Confirm with the user: `Remove mcp/<name> from config?`
3. Delete the array element
4. Check for orphaned references (e.g. a skill that was sourced from the
   plugin being removed) and warn:
   `~ skill <name> sources from plugin <plugin> which is being removed`

---

### Updating a version or tag

When the user says "pin to version X" or "update to latest":

For npx:
```json
"args": ["-y", "@scope/package@1.2.3"]
```

For uvx:
```json
"args": ["package==1.2.3"]
```

For Docker:
```json
"image": "mcp/server-name:1.2.3"
```

Replace `latest` with the specific version tag when pinning.

---

### Adding a marketplace

When the user mentions a new plugin registry or marketplace:

```json
{
  "name": "marketplace-alias",
  "url": "https://marketplace.example.com"
}
```

Verify the URL is reachable before writing. After adding, check if any
existing plugins or skills should reference this marketplace.

---

### Updating from chat instructions (examples)

These are typical user phrases and how to handle them:

| User says | Action |
|-----------|--------|
| "add Slack MCP" | Find Slack MCP package/image, add to `mcpServers`, add env vars to `.env.example` |
| "remove the figma server" | Find entry by name, confirm, delete |
| "the Terraform skill link is broken" | Search for correct repo, show fix, apply after confirm |
| "add terraform skill from GitHub" | Ask for repo URL or search, add to `skills` |
| "pin all MCPs to a specific version" | Iterate `mcpServers`, replace `latest` with discovered current version tag |
| "add my private registry" | Add to `marketplaces`, verify URL reachability |
| "install the ms-office plugin" | Add to `plugins` referencing its marketplace |
| "what MCPs are currently configured?" | Read and summarise `mcpServers` array |

---

## Reporting after edit

After a successful edit and validation pass, print a short summary:

```
── Config updated ──────────────────────
+ mcp/slack-server added (docker transport)
  ! Add SLACK_BOT_TOKEN to .env
✓ Validation passed
────────────────────────────────────────
```

If validation fails after the edit, restore the previous content and report:

```
✗ Edit rolled back — validation failed:
  ! mcp/slack-server: $SLACK_BOT_TOKEN not set in .env
  Add the variable to .env and run again.
```
