# Skills Management Reference

## What skills are

Skills are markdown files that give Claude Code specialised knowledge and
instructions for a domain. They are loaded into context when relevant and
persist across sessions as long as their files are present.

## Structure

A skill is a directory containing a `SKILL.md` file:

```
.claude/skills/<skill-name>/SKILL.md
```

Claude Code discovers skills by scanning `.claude/skills/` at session start.
Any directory containing a `SKILL.md` file is treated as an installed skill.

## Installing a skill from GitHub

```bash
git clone --depth 1 https://github.com/<owner>/<repo> .claude/skills/<name>
```

`--depth 1` clones only the latest commit, keeping the clone fast and lightweight.

To update an already-installed skill:

```bash
cd .claude/skills/<name> && git pull
```

## Checking if a skill is installed

```bash
test -f .claude/skills/<name>/SKILL.md && echo "installed" || echo "missing"
```

## Checking for drift

Drift occurs when the installed skill no longer matches its declared source.
Check the remote URL of the cloned repo:

```bash
cd .claude/skills/<name> && git remote get-url origin
```

Compare against the `repo` field in `claude-env.json`. If they differ,
report as `~` drift.

## Local skills

Skills with `source: "local"` are not cloned — they are authored in-place.
Verify they exist:

```bash
test -f <installPath>/SKILL.md && echo "present" || echo "missing"
```

## Plugin-sourced skills

Skills with `source: "plugin"` are provided by an installed Claude Code plugin.
They do not live in `.claude/skills/` and are not cloned from GitHub.
Claude Code makes them available automatically when the parent plugin is active.

To check if a plugin-sourced skill is available, verify the parent plugin is
installed (see `plugins-management.md`).

## Priority resolution

When the same skill is available from multiple sources, prefer in this order:

1. Plugin source — managed by Claude Code, always up to date
2. GitHub clone — manually installed, version pinned to last pull
3. Local path — manually authored, no external dependency

If a plugin source is unavailable (plugin not installed), fall back to GitHub.
Record the fallback in the apply output as `↩`.
