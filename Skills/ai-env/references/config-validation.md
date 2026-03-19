# Config Validation Reference

Run this validation pass before any apply. Report results using the
standard symbols: `✓` pass, `!` blocker, `~` warning.

A blocker (`!`) means that item cannot be applied. Collect all blockers,
report them together, and stop. Do not partially apply a broken config.

A warning (`~`) means something looks suspicious but apply can still
continue with reduced scope.

---

## 1. File existence

```bash
test -f ai-env.json && echo "present" || echo "missing"
```

If `ai-env.json` is missing, report `! ai-env.json not found` and stop.

---

## 2. JSON syntax

Parse the file:

```bash
python3 -c "import json,sys; json.load(open('ai-env.json'))" \
  && echo "valid JSON" || echo "syntax error"
```

If the file is not valid JSON, report `! ai-env.json: JSON syntax error`
and stop. Do not attempt field-level validation on a broken file.

---

## 3. Schema checks

After parsing, verify the following fields. All are optional in the schema,
but when present they must have the correct type.

| Field | Expected type | Blocker if wrong |
|-------|---------------|-----------------|
| `$schema` | string | no (warning only) |
| `marketplaces` | array | yes |
| `plugins` | array | yes |
| `skills` | array | yes |
| `mcpServers` | array | yes |
| `agents` | array | yes |

For each array element, check required sub-fields:

**marketplaces items:**
- `name` (string, required)
- `url` (string, required)

**plugins items:**
- `name` (string, required)
- `marketplace` (string, required) — must match a name in `marketplaces`
- `id` (string, required)
- `scope` (string, optional) — one of `project`, `user`, `local`; defaults to `project` if omitted

**skills items:**
- `name` (string, required)
- `source` (string, required) — must be one of: `github`, `plugin`, `local`
- `github` source: `repo` (string, required), `installPath` (string, optional)
- `plugin` source: `plugin` (string, required), `marketplace` (string, required)
- `local` source: `installPath` (string, required)

**mcpServers items:**
- `name` (string, required)
- `transport` (string, required) — must be one of: `docker`, `npx`, `uvx`, `local`
- `image` — required when transport is `docker`
- `package` — required when transport is `npx` or `uvx`
- `command` — required when transport is `local`
- `env` (object, optional) — values starting with `$` are placeholders

**agents items:**
- `name` (string, required)
- `path` (string, required)

---

## 4. Cross-reference checks

After schema validation:

**Plugin references:** Every skill with `source: "plugin"` must reference a
`plugin` name that exists in the `plugins` array. If missing, report:
`! skill <name>: references plugin <plugin-name> which is not declared`

**Marketplace references:** Every plugin must reference a `marketplace` that
exists in the `marketplaces` array. Same for plugin-sourced skills.

**installPath collisions:** No two skills should share the same `installPath`.
If a collision is found, report `~ installPath conflict: <path> used by <a> and <b>`.

---

## 5. Reachability checks

Run these only when network access is available.

**Marketplace URLs:**
```bash
curl -sf --max-time 5 -o /dev/null "<url>" \
  && echo "reachable" || echo "unreachable"
```
Unreachable marketplace → `~ marketplace <name>: URL not reachable (skipping dependent plugins/skills)`

**GitHub skill repos:**
```bash
curl -sf --max-time 5 -o /dev/null \
  "https://github.com/<owner>/<repo>" \
  && echo "exists" || echo "not found"
```
Not found → `! skill <name>: repo <owner>/<repo> not found or inaccessible`

**Docker images:**
```bash
docker pull --quiet <image> 2>&1 | tail -1
```
Pull failure → `! mcp <name>: Docker image <image> not found`

**npx packages:**
```bash
npm view <package> version 2>/dev/null || echo "not found"
```
Not found → `! mcp <name>: npm package <package> not found`

**uvx packages:**
```bash
pip index versions <package> 2>/dev/null | head -1 || echo "not found"
```
Not found → `! mcp <name>: PyPI package <package> not found`

---

## 6. Environment variable checks

For every `env` value starting with `$` in `mcpServers`:

1. Strip the `$` to get the variable name
2. Check `.env` for that key:
```bash
grep -q "^<VAR_NAME>=" .env 2>/dev/null && echo "found" || echo "missing"
```
3. Check the value is non-empty:
```bash
val=$(grep "^<VAR_NAME>=" .env | cut -d= -f2-)
[ -n "$val" ] && echo "non-empty" || echo "empty"
```

Missing or empty → `! env: $<VAR_NAME> required by mcp/<name> is not set`

If `.env` is absent entirely:
```
! .env file not found — create from .env.example or add required variables
```

---

## 7. Validation summary format

Print a summary block before proceeding:

```
── Validation ──────────────────────────
✓ JSON syntax valid
✓ Schema: 3 marketplaces, 1 plugin, 4 skills, 3 mcpServers, 0 agents
✓ Cross-references OK
~ marketplace acme-registry: URL not reachable — skipping dependent items
! mcp github-server: $GITHUB_TOKEN is not set in .env
! skill infra-tool: repo acme-org/infra-mcp not found
────────────────────────────────────────
2 blockers found — fix before applying
```

If there are no blockers, print `✓ Validation passed — proceeding with apply`.
