# Agents Management Reference

## What agents are

Agents (subagents) are Claude Code task specialists defined as markdown files
in `.claude/agents/`. Each file has a YAML frontmatter block that declares
the agent's identity, capabilities, and available tools. Claude Code uses the
`description` field to decide when to delegate a task to a specific agent.

---

## Agent file format

```markdown
---
name: agent-name
description: >
  One or more sentences describing when Claude Code should route a task
  to this agent. Be specific — this text is used for routing decisions.
tools:
  - bash
  - read
  - write
  - mcp__server-name__tool-name
model: claude-opus-4-5
---

## Agent instructions

The markdown body below the frontmatter contains the agent's system prompt
and working instructions.
```

### Frontmatter fields

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | string | Unique identifier. Should match the filename without extension. |
| `description` | yes | string | Routing description. Used by Claude Code to select this agent. |
| `tools` | no | list of strings | If absent, agent inherits all tools from the parent session. |
| `model` | no | string | Override the model for this agent. If absent, uses session default. |

---

## Validation checklist

For each agent file declared in `ai-env.json`, run the following checks.

### 1. File existence

```bash
test -f <path> && echo "present" || echo "missing"
```

Missing → `! agent/<name>: file not found at <path>`

### 2. YAML frontmatter parseable

Check that the file starts with `---` and has a closing `---` delimiter:

```bash
head -1 <path> | grep -q "^---" && echo "has frontmatter" || echo "missing frontmatter"
```

Extract and parse the frontmatter block:

```bash
python3 - <<'EOF'
import sys, re
content = open('<path>').read()
m = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
if not m:
    print("ERROR: no frontmatter block found")
    sys.exit(1)
import yaml
try:
    data = yaml.safe_load(m.group(1))
    print("OK:", list(data.keys()))
except yaml.YAMLError as e:
    print("ERROR:", e)
    sys.exit(1)
EOF
```

Parse error → `! agent/<name>: YAML frontmatter is invalid — <error>`

### 3. Required fields present

After parsing, check for `name` and `description`:

```python
if 'name' not in data:
    print("ERROR: missing required field 'name'")
if 'description' not in data:
    print("ERROR: missing required field 'description'")
```

Missing field → `! agent/<name>: missing required frontmatter field '<field>'`

### 4. Name matches filename

The `name` field in frontmatter should match the filename (without `.md`):

```python
import os
expected = os.path.basename('<path>').replace('.md', '')
if data.get('name') != expected:
    print(f"WARNING: name '{data['name']}' does not match filename '{expected}'")
```

Mismatch → `~ agent/<name>: frontmatter name "<value>" does not match filename`

### 5. Tools — built-in names

Claude Code built-in tool names (case-sensitive):

```
bash, read, write, edit, glob, grep, ls, todoread, todowrite,
webfetch, websearch, task, notebookedit, exitplanmode
```

For each entry in `tools` that does not start with `mcp__`, check it against
this list. Unknown tool name → `~ agent/<name>: unknown built-in tool '<tool>'`

### 6. Tools — MCP references

MCP tool references follow this pattern:

```
mcp__<server-name>__<tool-name>
```

Where `<server-name>` must match the `name` of a server in the effective MCP
list (built from `mcpServers` + plugins + skills).

For each `mcp__<server-name>__<tool-name>` entry in `tools`:

1. Extract `<server-name>` (the part between the first and second `__`)
2. Check whether that server name exists in the effective MCP list

```python
# Assuming effective_mcp_list is a set of server names already built
tool = "mcp__my-server__some-tool"
parts = tool.split("__")
if len(parts) >= 2:
    server_name = parts[1]
    if server_name not in effective_mcp_list:
        print(f"ERROR: tool '{tool}' references server '{server_name}' which is not in the effective MCP list")
```

Unknown server → `! agent/<name>: tool references MCP server '<server>' which is not configured`

### 7. Model field (if present)

If `model` is set, check it is a non-empty string. Claude Code will reject
empty model values at runtime. A valid model string looks like:
`claude-opus-4-5`, `claude-sonnet-4-5`, `claude-haiku-4-5-20251001`.

Content validation cannot confirm the model string is a valid Anthropic model
identifier without an API call — report as `~` warning if the format looks
unusual (e.g. contains spaces, is very short):

```python
model = data.get('model', '')
if model and (len(model) < 5 or ' ' in model):
    print(f"WARNING: model value '{model}' looks invalid")
```

Suspicious model string → `~ agent/<name>: model value "<value>" looks invalid`

---

## Reporting agent validation results

Include agent validation in the apply output:

```
Agents
  ✓ backend-dev     present   frontmatter valid, 4 tools checked
  ! frontend-dev    missing   .claude/agents/frontend-dev.md not found
  ~ data-analyst    warning   tool mcp__reporting__query references unconfigured server 'reporting'
  ! pipeline-bot    invalid   YAML frontmatter parse error: line 3: mapping values not allowed
```

---

## Scaffolding a missing agent

When an agent file is missing, offer to create a minimal scaffold so the
user has a starting point:

```markdown
---
name: <name>
description: >
  Describe when Claude Code should use this agent.
tools:
  - bash
  - read
  - write
---

## Instructions

Describe what this agent should do and how.
```

Ask the user to confirm before writing the file. The scaffold is a starting
point only — the `description` and instructions must be filled in by the user.
