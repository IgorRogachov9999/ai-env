# Lockfile Reference

The lockfile (`ai-env.lock.json`) records the exact state of the
environment after each successful apply. It is the baseline for
drift detection on subsequent runs and allows team members to reproduce
the same environment.

Commit `ai-env.lock.json` to version control.
Do NOT commit `.mcp.json` or `.env` — they contain resolved secrets.

---

## Format

```json
{
  "version": 1,
  "appliedAt": "2025-01-15T10:30:00Z",
  "skills": {
    "skill-name": {
      "source": "github",
      "repo": "owner/repo",
      "installPath": ".claude/skills/skill-name",
      "commit": "abc1234def5678"
    },
    "plugin-skill-name": {
      "source": "plugin",
      "plugin": "plugin-name"
    }
  },
  "mcpServers": {
    "server-name": {
      "transport": "docker",
      "image": "mcp/server-name:1.2.3",
      "digest": "sha256:a1b2c3..."
    },
    "another-server": {
      "transport": "npx",
      "package": "@scope/mcp-package",
      "resolvedVersion": "2.0.1"
    }
  },
  "plugins": {
    "plugin-name": {
      "marketplace": "marketplace-alias",
      "id": "plugin-id",
      "installedAt": "2025-01-15T10:30:00Z"
    }
  },
  "agents": {
    "agent-name": {
      "path": ".claude/agents/agent-name.md",
      "hash": "sha256:d4e5f6..."
    }
  }
}
```

---

## Writing the lockfile

After a successful apply, collect version info for each installed
resource and write `ai-env.lock.json` atomically (write to a temp file,
then rename).

### Skills — GitHub source

Record the HEAD commit hash of the cloned repo:

```bash
cd <installPath> && git rev-parse HEAD
```

### Skills — plugin source

Record only the parent plugin name. Plugin versions are managed by the
marketplace and not independently pinnable.

### MCP servers — Docker transport

Record the image tag and, if available, the resolved content digest:

```bash
docker inspect --format='{{index .RepoDigests 0}}' image:tag 2>/dev/null
```

If the image has not been pulled yet (first run), record the tag as-is and
omit `digest`. The digest will be populated on the next apply after
the image is pulled.

### MCP servers — npx transport

Resolve and record the installed version:

```bash
npm view @scope/package version 2>/dev/null
```

### MCP servers — uvx transport

```bash
uvx package-name --version 2>/dev/null | head -1
```

### Plugins

Record the installation timestamp. Plugin version numbers are not reliably
exposed by marketplace APIs; timestamp is the best available baseline.

### Agents

Record the SHA-256 hash of the file contents:

```bash
sha256sum .claude/agents/agent-name.md | cut -d' ' -f1
```

---

## Reading the lockfile for drift detection

On apply start, if `ai-env.lock.json` exists, compare the
locked state against the current filesystem before comparing against
`ai-env.json`. This catches changes that happened outside the skill
(e.g. a team member ran `git pull` on a skill directory, or edited an agent).

### Skill drift

```bash
cd <installPath> && git rev-parse HEAD
```

If the current commit differs from the locked commit → report as `~` drift:
`~ skill/<name>: commit changed since last apply (locked: abc1234, current: def5678)`

### Agent drift

```bash
sha256sum .claude/agents/agent-name.md | cut -d' ' -f1
```

If the hash differs from the locked hash → report as `~` drift:
`~ agent/<name>: file changed since last apply`

### MCP server drift

For Docker, check if the image tag still resolves to the same digest:

```bash
docker inspect --format='{{index .RepoDigests 0}}' image:tag 2>/dev/null
```

If the digest changed → report as `~` drift (a new image was pulled under
the same tag).

---

## Lockfile conflicts in version control

If two team members apply independently and both update the lockfile,
a merge conflict will occur. To resolve:

1. Accept either version (both are valid snapshots)
2. Re-run apply to regenerate the lockfile from the current state
3. Commit the regenerated lockfile

---

## When the lockfile does not exist

If `ai-env.lock.json` is absent (first run, or deleted), skip drift
detection and proceed with a full apply from scratch. Write the
lockfile after successful completion.
