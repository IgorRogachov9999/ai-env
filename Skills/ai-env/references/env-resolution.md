# Environment Variable Resolution Reference

## .env file format

```bash
# Comments start with #
SERVICE_API_KEY=your-api-key-here
SERVICE_BASE_URL=https://your-org.example.com
SERVICE_EMAIL=you@example.com
```

Rules:
- One variable per line: `KEY=value`
- No spaces around `=`
- Values do not need quotes unless they contain spaces or special characters
- Lines starting with `#` are comments and are ignored
- Empty lines are ignored

## Placeholder syntax in ai-env.json

In `ai-env.json`, env var values use `$` prefix as placeholders:

```json
"env": {
  "SERVICE_API_KEY": "$SERVICE_API_KEY",
  "SERVICE_BASE_URL": "$SERVICE_BASE_URL"
}
```

During apply, every `$VARIABLE` placeholder is resolved by looking up
the variable name (without `$`) in `.env`. The resolved value is written into
`.mcp.json`. Never write `$VARIABLE` placeholders into `.mcp.json` — they
will not be expanded by Claude Code.

## Resolution algorithm

For each `$VAR` placeholder in `ai-env.json`:

1. Strip the `$` prefix to get the variable name
2. Read `.env` and look up the name
3. If found and non-empty → use the resolved value
4. If the key exists but value is empty → treat as missing
5. If the key is absent → treat as missing

If any variable is missing, collect all missing names, report them as `!`
items, and stop before writing `.mcp.json`. Do not partially write the file.

## .env.example

Projects should commit a `.env.example` file with all required keys and empty
values. This documents what is needed without exposing secrets:

```bash
# Copy to .env and fill in the values
SERVICE_API_KEY=
SERVICE_BASE_URL=
SERVICE_EMAIL=
```

When `.env` is missing, suggest the user run:

```bash
cp .env.example .env
# then fill in the values
```

## Security rules

- Never commit `.env` — add it to `.gitignore`
- Never commit `.mcp.json` — it contains resolved values from `.env`
- Never log or print resolved secret values during apply
- In the apply output, show variable names but not their values:
  `✓ SERVICE_API_KEY resolved` not `✓ SERVICE_API_KEY = sk-abc123`
