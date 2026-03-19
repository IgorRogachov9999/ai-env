---
name: ai-env
description: >
  Manages the Claude Code AI development environment using a declarative
  desired-state model. Reads ai-env.json, compares it to the actual
  environment, and applies the desired state — connecting marketplaces,
  installing plugins, installing skills, generating .mcp.json from MCP server
  declarations, and verifying agent files. Idempotent: safe to run at any
  time, only acts on what has drifted from the desired state.

  Use this skill whenever ai-env.json is present and the user asks to
  set up, bootstrap, sync, or refresh the AI environment. Also trigger at
  the start of a session when .mcp.json is missing or ai-env.json has
  changed since it was last applied. Trigger when the user asks to add,
  remove, or fix anything in the environment config. Trigger for read-only
  queries about current state or status. Trigger for dry-run requests to
  preview what would change. Trigger when the user asks to find or
  recommend plugins for their workflow.
---

## Overview

`ai-env.json` is the single source of truth for the project's AI
development environment. It declares five resource types:

| Resource | Description |
|----------|-------------|
| `marketplaces` | Plugin marketplaces and package registries to connect to. |
| `plugins` | Plugins to install from connected marketplaces. |
| `skills` | Claude Code skills — sourced from a plugin, GitHub, or a local path. |
| `mcpServers` | MCP servers to configure in `.mcp.json`. |
| `agents` | Subagent files expected to exist in `.claude/agents/`. |

Read `ai-env.json` before doing anything else. If the file does not
exist, inform the user and stop.

A `schema.json` file is included in this skill. Users can reference it in
their `ai-env.json` for IDE validation and autocomplete:

```json
{ "$schema": ".claude/skills/ai-env/schema.json" }
```

---

## Help

When the user asks for `help`, `--help`, `usage`, or "what can you do" in the
context of this skill, print the following command reference and stop — do not
apply anything:

```
ai-env — declarative Claude Code environment manager

COMMANDS
  (default)   Read ai-env.json and apply desired state to the environment.
              Installs missing resources, removes orphaned ones, writes .mcp.json.

  plan        Dry-run: show what would change without applying anything.
              Phrases: "plan", "dry-run", "what would change", "preview changes"

  status      Read-only snapshot of the current environment state.
              Phrases: "status", "what's installed", "list MCPs", "show environment"

  edit        Apply a natural-language change to ai-env.json.
              Phrases: "add X", "remove Y", "fix the link for Z", "update version"

  plugins     Find and recommend plugins for a described workflow.
              Phrases: "what plugins do I need for X", "find plugins for Y"

  help        Show this message.
              Phrases: "help", "--help", "usage", "what commands do you have"

CONFIG FILE
  ai-env.json — desired state (commit this)
  ai-env.lock.json — exact installed versions (commit this)
  .mcp.json — generated, do not commit
  .env — secrets, do not commit

RESOURCE TYPES
  marketplaces   Plugin registries to connect to
  plugins        Plugins from marketplaces (scope: project | user | local)
  skills         Claude Code skills (source: plugin | github | local)
  mcpServers     MCP servers → written to .mcp.json (transport: docker > npx > uvx)
  agents         Subagent .md files validated in .claude/agents/

SCHEMA
  Add "$schema": ".claude/skills/ai-env/schema.json" to ai-env.json
  for IDE validation and autocomplete.
```

---

## Validation

Before applying or editing the config, always run a validation pass.
See `references/config-validation.md` for the full checklist.

The short version:
1. File exists and is valid JSON
2. Required fields have the correct types
3. Cross-references between sections are consistent
4. Referenced repos, images, and packages are reachable
5. All `$VARIABLE` placeholders have values in `.env`

If there are blockers, stop and report them. Do not apply a broken config.

---

## Editing the config

When the user asks to change the environment configuration in chat (add a
server, fix a link, remove a skill, etc.), use the edit workflow from
`references/config-editing.md`:

1. Read the current `ai-env.json`
2. Plan and explain the change
3. Apply the edit
4. Validate the result
5. Report outcome

Always validate after editing. Roll back on validation failure.

---

## Plan mode (dry-run)

Triggered when the user asks to preview changes without applying them:
"plan", "dry-run", "what would change", "show me what would happen".

1. Run the full validation pass (same as normal)
2. Compare desired state (ai-env.json) against actual state — same logic
   as a normal apply, but **do not write any files or install anything**
3. Read `ai-env.lock.json` if it exists and include locked versions in
   the comparison
4. Output the planned changes using the same format as normal apply output,
   but with a `(planned)` suffix on each action line:

```
ai-env: plan (no changes applied)

Skills
  + terraform     would clone   owner/repo → .claude/skills/terraform  (planned)
  ✓ docx          active        (plugin: ms-office-suite)

MCP Servers
  + my-server     would add     to .mcp.json                            (planned)
  ✓ another       present       (no change)

Summary: 2 would be added · 0 would be removed · 0 drift · 0 requiring action
Run without dry-run to apply.
```

Plan mode never writes files. It is always safe to run.

---

## Status

Triggered by read-only queries: "what's installed", "show environment
status", "what's currently configured", "list MCPs".

Status is like plan mode but scoped to the current actual state only —
it does not compute diffs or propose changes.

1. Read `ai-env.json` (desired state)
2. Read `ai-env.lock.json` if it exists (last applied state)
3. Check actual filesystem state
4. Output a snapshot — no diffs, no proposals:

```
ai-env: status

Skills (2 installed)
  ✓ terraform     github    owner/repo @ abc1234    .claude/skills/terraform
  ✓ docx          plugin    ms-office-suite

MCP Servers (3 in .mcp.json)
  ✓ my-server     docker    mcp/my-server:1.2.3
  ✓ another       npx       @scope/package @ 2.0.1
  ~ orphan-srv    docker    (in .mcp.json but not in ai-env.json)

Plugins (1 installed)
  ✓ ms-office-suite    anthropic marketplace   [project]

Agents (2 declared)
  ✓ backend-dev    .claude/agents/backend-dev.md
  ! frontend-dev   .claude/agents/frontend-dev.md  (file missing)

Last applied: 2025-01-15 10:30 UTC
```

Status never writes files.

---

## Apply

For each resource type, determine the current state first, then act only on
what differs from the desired state. Resources already in desired state are
never modified.

Process resource types in dependency order: marketplaces → plugins → skills →
mcpServers → agents. A later stage may depend on an earlier one being ready
(e.g. a skill sourced from a plugin requires that plugin to be installed first).

### Marketplaces

A marketplace is in desired state when it is accessible and available as a
source for the current session.

| Current state | Action |
|---------------|--------|
| Marketplace reachable and connected | No action |
| Marketplace not connected | Connect or prompt the user to connect if manual auth is required |

`npm`, `pypi`, and `docker` are package registries used implicitly when
installing MCP servers — no explicit connection step is needed for these.

Plugin marketplaces (e.g. `anthropic`) may require the user to authenticate.
If authentication is needed and cannot be completed automatically, report it
as a `!` item and continue with the remaining resources.

### Plugins

A plugin is in desired state when it is installed and active for the session.

| Current state | Action |
|---------------|--------|
| Plugin installed and active | No action |
| Plugin not installed | Install from the declared `marketplace` using the declared `id` |
| Declared marketplace not connected | Report as blocked, skip — do not attempt install |

**Scope:** Each plugin entry may declare a `scope` field:

| Value | Meaning |
|-------|---------|
| `"project"` | Install at project scope (default if omitted) |
| `"user"` | Install for the current user only |
| `"local"` | Install locally, not committed to version control |

When `scope` is not specified, always install at project level.
Pass the appropriate scope flag to the marketplace install command based on
this value.

When a plugin is installed, note which skills it provides. These become
available as `source: "plugin"` options for the skills stage.

### Skills

Skills support three source types. **Prefer plugin sources over GitHub or
local** — if a skill is available via an already-installed plugin, use that
rather than cloning from GitHub even if both are declared.

**Source: `"plugin"`**

| Current state | Action |
|---------------|--------|
| Parent plugin installed, skill active | No action |
| Parent plugin not installed | Report as blocked — plugin must be installed first |

**Source: `"github"`**

A skill is in desired state when its `installPath` directory exists and is
non-empty.

| Current state | Action |
|---------------|--------|
| `installPath` exists and non-empty | No action |
| `installPath` absent | Clone: `git clone --depth 1 https://github.com/{repo} {installPath}` |
| `installPath` exists but source has changed | Report as drift — do not re-clone unless explicitly asked |

**Source: `"local"`**

| Current state | Action |
|---------------|--------|
| Path exists | No action |
| Path absent | Report as missing — no automatic action |

**Priority rule:** When resolving a skill, check sources in this order:
1. Plugin (from a connected marketplace)
2. GitHub clone
3. Local path

If a skill declares `source: "plugin"` but its plugin is not installed, fall
back to a GitHub source for the same skill if one is also declared, and note
the fallback in the output.

### MCP Servers

**Transport priority:** When a server is available via multiple transports,
prefer Docker over npx over uvx. Docker images are self-contained and avoid
runtime version drift. Fall back to npx or uvx only when no Docker image is
available for the server.

Desired state: `.mcp.json` exists in the project root with a correct entry
for every required server, with no duplicates.

Before applying, build the **effective MCP list** by merging all sources
in priority order — higher priority wins when the same server name appears
in multiple sources:

1. `mcpServers` declared explicitly in `ai-env.json` — highest priority
2. MCP servers bundled by installed `plugins` — lower priority
3. MCP servers bundled by installed `skills` — lowest priority

Deduplication rule: if the same server name appears in more than one source,
keep only the highest-priority entry and discard the rest. Note discarded
duplicates in the output as `~` drift entries so the user is aware.

Once the effective list is built, update `.mcp.json` to match it:

| Current state | Action |
|---------------|--------|
| Entry present with correct config | No action |
| Entry absent | Add it |
| Entry present with incorrect config | Update it |
| Entry in `.mcp.json` not in effective list | Remove it (orphaned) |

Rewrite `.mcp.json` only when at least one change is made.

**Drift detection against lockfile:** Before comparing desired vs actual state,
read `ai-env.lock.json` if it exists and check whether any installed
resources have changed since the last apply. See
`references/lockfile.md` for the per-resource-type detection commands.
Report externally-changed resources as `~` drift before processing changes.

**Environment variable resolution:** Values prefixed with `$` are placeholders
resolved from `.env` in the project root. If `.env` is missing, or if any
required variable is absent, list every unresolved variable and stop. Do not
write a `.mcp.json` containing blank or unresolved values.

**Transport output formats for `.mcp.json`:**

```json
// npx
{ "command": "npx", "args": ["-y", "<package>"], "env": {} }

// uvx
{ "command": "uvx", "args": ["<package>"], "env": {} }

// docker
{ "command": "docker", "args": ["run", "--rm", "-i", "--env", "KEY=VALUE", "<image>"] }
```

### Agents

An agent is in desired state when its file exists on disk **and** its
content is valid. See `references/agents-management.md` for the full
validation checklist.

**Existence check:**

| Current state | Action |
|---------------|--------|
| File exists | Proceed to content validation |
| File absent | Offer to scaffold a minimal template, report as `!` |

**Content validation** (run for every file that exists):

1. YAML frontmatter is present and parseable
2. Required fields `name` and `description` are present
3. `name` matches the filename (warning only if not)
4. Built-in tool names in `tools` are valid
5. MCP tool references (`mcp__<server>__<tool>`) point to servers that exist
   in the effective MCP list built in the previous stage
6. `model` value (if set) looks like a valid model string

Content validation failures are reported as `!` (invalid frontmatter, missing
required field, unknown MCP server) or `~` (name mismatch, suspicious model
string). They do not block other agents from being validated.

If the `agents` array is empty, skip this section silently.

---

## Output

After apply, print a structured summary. Use the following symbols:

```
✓  in desired state — no action taken
+  added — resource was missing and has been created or installed
-  removed — resource was orphaned and has been removed
~  drift — state differs but was not automatically changed (includes deduplicated MCP entries)
!  action required — cannot be resolved without human input
↩  fallback — plugin unavailable, fell back to alternative source
```

Format:

```
ai-env: apply complete

Marketplaces
  ✓ <name>   connected
  ! <name>   authentication required — connect manually then re-run

Plugins
  ✓ <name>   installed   (<marketplace>)  [<scope>]
  + <name>   installed   (<marketplace>)  [<scope>]
  ! <name>   blocked     (marketplace not connected)

Skills
  ✓ <name>   active      (plugin: <plugin-name>)
  + <name>   added       (cloned from <repo>)
  ↩ <name>   fallback    (plugin unavailable — cloned from <repo>)
  ~ <name>   drift       (<reason>)
  ! <name>   blocked     (<reason>)

MCP Servers
  ✓ <name>   present     (no change)
  + <name>   added       (.mcp.json updated)
  - <name>   removed     (orphaned)
  ~ <name>   deduped     (provided by plugin <x> and plugin <y> — kept <x>, discarded <y>)

Agents
  ✓ <name>   present     (<file>)
  ! <name>   missing     (<file> — create manually)

Secrets
  ✓ <n>/<total> resolved from .env
  ! <VAR_NAME> — not found in .env

Summary: <n> added · <n> removed · <n> drift · <n> requiring action
```

After printing the summary, if there were no blockers and at least one
resource was added or removed, update `ai-env.lock.json` with the new
state. See `references/lockfile.md` for the per-resource-type version
collection commands.

---

## Idempotency

Running apply is safe at any time. When the environment already
matches the desired state, no changes are made and the output shows only
`✓` entries. Running the skill multiple times produces the same result.

---

## Security notes

After writing `.mcp.json` for the first time, verify that both `.mcp.json`
and `.env` are present in `.gitignore`. These files contain resolved secret
values and must not be committed to version control.

---

## Reference files

Read these files when you need implementation details for a specific area.
Do not load all of them upfront — read only what the current apply step requires.

| File | Read when |
|------|-----------|
| `references/config-validation.md` | Validating `ai-env.json` before any apply or edit |
| `references/config-editing.md` | Updating `ai-env.json` from user instructions (add, remove, fix) |
| `references/lockfile.md` | Reading or writing `ai-env.lock.json`; drift detection against locked state |
| `references/mcp-docker.md` | Processing any server with `transport: "docker"` |
| `references/mcp-npm.md` | Processing any server with `transport: "npx"` |
| `references/mcp-uvx.md` | Processing any server with `transport: "uvx"` |
| `references/mcp-config-format.md` | Writing or validating `.mcp.json` |
| `references/skills-management.md` | Installing, checking, or detecting drift in skills |
| `references/plugins-management.md` | Installing plugins or resolving plugin-sourced skills/MCPs |
| `references/plugin-discovery.md` | Finding and selecting plugins when the user asks for recommendations |
| `references/env-resolution.md` | Resolving `$VARIABLE` placeholders or diagnosing missing secrets |
| `references/agents-management.md` | Validating agent frontmatter, tool references, and scaffolding missing files |
