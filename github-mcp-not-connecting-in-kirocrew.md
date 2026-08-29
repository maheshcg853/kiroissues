# GitHub MCP Server Not Connecting in KiroCrew (works in Kiro CLI)

**Date:** 2026-08-29
**Component:** KiroCrew desktop app / MCP configuration
**Status:** Resolved
**Severity:** Medium (single MCP server unavailable in one surface)

## Summary

The `github` MCP server connected fine in Kiro CLI but failed in the KiroCrew
desktop app. The dashboard showed GitHub as "not connected," and validation
surfaced a Dynamic Client Registration (DCR) failure / HTTP 400. Root cause was
an unresolved environment-variable placeholder in the GitHub server's auth
header when launched by the desktop app. Fixed by writing the literal token into
KiroCrew's own MCP config source file.

## Symptoms

- Kiro CLI: `github` MCP server connected, tools available.
- KiroCrew desktop: `github` shows as not connected / failing.
- Error captured when loading the `kirocrew` agent via the CLI with the env var
  unset:
  ```
  Dynamic registration failed: Registration failed: Dynamic client registration not supported
  Not all mcp servers loaded.
  ```
- Restarting the KiroCrew desktop app (even with all-new PIDs) did not fix it.

## Root Cause

The GitHub MCP server is a remote HTTP server authenticated with a header:

```
Authorization: Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}
```

The `${...}` is an environment-variable placeholder that is only substituted if
the launching process has that variable in its environment.

- **Kiro CLI** runs from the interactive shell, which exports
  `GITHUB_PERSONAL_ACCESS_TOKEN` (from `~/.zshrc` / `~/.bashrc`), so the
  placeholder resolved correctly.
- **KiroCrew desktop** launches its gateway from the `/Applications/KiroCrew.app`
  bundle (not the shell), so the variable was absent. The header was sent
  literally as `Bearer ${GITHUB_PERSONAL_ACCESS_TOKEN}`.

With no usable bearer token, `kiro-cli` treated the HTTP MCP server as needing
OAuth and attempted Dynamic Client Registration (DCR). GitHub's MCP endpoint
(`https://api.githubcopilot.com/mcp/`) does not support DCR, which produced the
failure / 400.

### Why the fix initially didn't "stick"

There are **three** relevant config files. The desktop app regenerates the live
agent config on every launch from its **own** MCP config, not from the CLI's:

| File | Role |
|------|------|
| `~/.kiro/settings/mcp.json` | Kiro CLI's global MCP config |
| `~/.kiro/crew/mcp.json` | **KiroCrew's own MCP config — the regeneration source** |
| `~/.kiro/agents/kirocrew.json` | Live agent config, overwritten by the app on each launch |

Early edits to `kirocrew.json` and `settings/mcp.json` were repeatedly undone
because, on each app launch, KiroCrew copied the old placeholder-form header from
`~/.kiro/crew/mcp.json` back into `kirocrew.json`.

## Diagnosis Steps

1. Direct `curl` to the GitHub MCP endpoint with the token returned HTTP 200 and
   a valid MCP handshake → token and endpoint are valid.
2. Confirmed the token was a valid 40-char classic PAT (`ghp_...`).
3. Loaded the `kirocrew` agent via `kiro-cli chat --no-interactive --agent kirocrew`
   with the env var unset → reproduced the DCR error.
4. Compared the `github` header across the three config files; found the
   placeholder form still present in `~/.kiro/crew/mcp.json`.
5. Located the true source by searching for the literal placeholder string:
   ```
   grep -rl 'GITHUB_PERSONAL_ACCESS_TOKEN}' ~/.kiro/
   ```

## Resolution

Replaced the placeholder with the literal token value in the GitHub server's
`Authorization` header, most importantly in KiroCrew's own source config so it
persists across app relaunches:

```
Authorization: Bearer ghp_<actual-token>
```

Files updated:
- `~/.kiro/crew/mcp.json`  (the regeneration source — the critical one)
- `~/.kiro/agents/kirocrew.json`  (live config, for immediate effect)
- `~/.kiro/settings/mcp.json`  (CLI global config, kept in sync)

With a literal token present, `kiro-cli` uses it directly and never attempts the
OAuth/DCR path.

### Verification

- Loaded the `kirocrew` agent with `GITHUB_PERSONAL_ACCESS_TOKEN` **unset**
  (the desktop app's condition): GitHub connected and exposed ~59 tools.
- After a gateway restart, the header in `kirocrew.json` stayed as the literal
  token (not re-reverted), confirming the true source was fixed.
- No further DCR / 400 errors in the gateway logs.
- User confirmed GitHub shows connected in the KiroCrew dashboard.

## Key Learnings / Things to Stay Aware Of

1. **Authoritative file for KiroCrew MCP servers is `~/.kiro/crew/mcp.json`.**
   Edit GitHub/MCP settings there, not just in `kirocrew.json` (which the app
   regenerates on launch).
2. **`${VAR}` placeholders only resolve for processes that inherit the variable.**
   GUI apps launched from the dock/Finder generally do NOT inherit shell-exported
   env vars. Use a literal value (or a recognized secret store) for app-launched
   surfaces.
3. **Token rotation:** the PAT is stored as plaintext in three config files (all
   user-only dirs). When rotating, update `~/.kiro/crew/mcp.json` first, then
   relaunch KiroCrew; update the other two to keep CLI/agent in sync.
4. **KiroCrew `.env` is a scoped channel-credential store**, not a general secret
   store. It only agent-isolates a fixed allow-list of keys (Slack, Discord,
   Jira, `KIRO_API_KEY`, etc.). `GITHUB_PERSONAL_ACCESS_TOKEN` is NOT on that
   list — it logs an "unrecognised credential" warning and is not agent-isolated.
5. **Restart semantics:** closing the KiroCrew window does not stop the gateway.
   Use full Quit (Cmd+Q), or `kirocrew restart` / `kirocrew stop`. Config changes
   only take effect when the gateway and its ACP child processes respawn.

## Open / Optional Follow-ups

- Investigate whether an env-var-only approach can be made to work via
  `mcp_gateway.forward_declared_env` so the token can live only in `.env` and the
  header can stay a `${...}` placeholder (not pursued — earlier test still failed
  with DCR; behavior of `.env` → HTTP-header expansion is not documented).
- Optionally remove the now-redundant `GITHUB_PERSONAL_ACCESS_TOKEN` from
  `~/.kiro/crew/.env` to reduce copies of the secret.
- Optionally revert `~/.kiro/settings/mcp.json` to the `${...}` form to keep the
  CLI's config free of the plaintext token (the shell provides it there).

## Related Commands

```bash
# Identify who is logged in
kiro-cli whoami

# KiroCrew health check (reports kiro-cli binary + auth + MCP status)
kirocrew doctor

# Restart the KiroCrew gateway (respawns ACP children)
kirocrew restart

# Find which config file holds an unexpanded placeholder
grep -rl 'GITHUB_PERSONAL_ACCESS_TOKEN}' ~/.kiro/
```
