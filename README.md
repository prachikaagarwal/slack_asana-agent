# Slack → Asana Daily Sync

Scans customer Slack channels (`#airops-*`, `#ext-*`, `#ext-airops-*`), extracts CMS integration issues per client, detects resolution from your replies and reactions, and writes one consolidated comment per Asana ticket. Also flags stale tickets and saves a `dashboard_data.json` for reporting.

**Entry point:** `SKILL.md` — trigger by saying "sync slack to asana" or "scan customer channels" in Claude Code.

---

## Prerequisites

This skill runs entirely through MCP servers — no API tokens or `.env` secrets needed. You need two MCPs connected: **Slack** and **Asana**.

---

## 1. Connect Slack MCP

Slack provides a hosted MCP server with OAuth — no tokens to manage.

```bash
claude mcp add --transport http slack https://mcp.slack.com/mcp
```

Or add manually to `~/.claude/claude.json`:

```json
{
  "mcpServers": {
    "slack": {
      "type": "http",
      "url": "https://mcp.slack.com/mcp"
    }
  }
}
```

The first time the skill runs, Claude Code will open a browser for the Slack OAuth flow. Approve the following scopes:

- `search:read` — search messages across channels
- `channels:history` — read public channel messages
- `groups:history` — read private channel messages
- `users:read` + `users:read.email` — resolve user profiles
- `reactions:read` — detect emoji reactions (used for resolution detection)
- `files:read` — detect file attachments

> If your workspace requires admin approval for new apps, ask your Slack admin to approve the Slack MCP app.

---

## 2. Connect Asana MCP

Use the V2 URL — the V1 beta is deprecated.

```bash
claude mcp add --transport http asana https://mcp.asana.com/v2/mcp
```

Or add manually to `~/.claude/claude.json`:

```json
{
  "mcpServers": {
    "asana": {
      "type": "http",
      "url": "https://mcp.asana.com/v2/mcp"
    }
  }
}
```

To authenticate with OAuth:

1. Go to [Asana Developer Console](https://app.asana.com/0/my-apps)
2. Create a new app — select **MCP app** as the app type
3. Set redirect URL to: `http://localhost:8080/callback`
4. Run:

```bash
claude mcp add --transport http \
  --client-id YOUR_CLIENT_ID \
  --client-secret YOUR_CLIENT_SECRET \
  --callback-port 8080 \
  asana https://mcp.asana.com/v2/mcp
```

The browser will open for OAuth on first use. Credentials are stored in your system keychain.

---

## 3. Verify Both MCPs Are Connected

In Claude Code:

```
/mcp
```

Both `slack` and `asana` should show as connected. If either is disconnected, re-run the `claude mcp add` command for that server.

---

## 4. Configure Runtime Settings

```bash
cp .env.example .env
```

Edit `.env`:

| Variable | Required | Description |
|---|---|---|
| `ASANA_PROJECT_ID` | Yes | GID of your Asana project (default: `1213494960962920`) |
| `USER_NAME` | Recommended | Your name — used to resolve your Slack ID at runtime |
| `LOOKBACK_DAYS` | No | Days back to scan (default: `3`) |
| `STALE_TICKET_SKIP` | No | Comma-separated ticket name substrings to skip for stale nudges |
| `DASHBOARD_OUTPUT_PATH` | No | Absolute path to write `dashboard_data.json` |

`SLACK_USER_ID` and `ASANA_USER_ID` are resolved automatically from your MCP connections — leave blank.

---

## 5. Run the Skill

Just say it in Claude Code:

```
sync slack to asana
```

Other triggers: "scan customer channels", "check what customers need", "what did customers say this week".

The skill will:
1. Scan all `#airops-*`, `#ext-*`, and `#ext-airops-*` customer channels for the last `LOOKBACK_DAYS` days
2. Extract integration issues per customer (pain point, current status, ideal status, resolution)
3. Match each customer to their Asana ticket
4. Write one `<!-- customer-slack-sync -->` comment per ticket (skips if already synced this run)
5. Post `<!-- customer-slack-stale-nudge -->` on tickets with no update in 3+ days
6. Save `dashboard_data.json`
7. Print a summary

---

## Project Structure

```
SKILL.md          — full skill definition (entry point)
CLAUDE.md         — project rules and runtime config reference
.env              — your local settings (no secrets)
.env.example      — settings template
workflows/        — detailed per-step reference docs (01–10)
output/{date}/    — daily run outputs (slack/ and asana/ subdirs)
README.md         — this file
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Slack MCP not finding private channels | Ensure `groups:history` scope is approved in OAuth |
| Asana MCP returns no tasks | Confirm `ASANA_PROJECT_ID` is correct in `.env` |
| OAuth flow doesn't open | Run `claude mcp remove slack` then re-add |
| Asana URL errors | Ensure config URL is `https://mcp.asana.com/v2/mcp` (V2) |
| Skill doesn't trigger | Run `/mcp` — both servers must show as connected |
