# Slack → Asana Daily Sync

Scans customer Slack channels, extracts CMS integration issues per client, detects resolution, and writes consolidated comments to Asana tickets.

**Entry point:** `SKILL.md` — invoke via `/customer-slack-to-asana` or by asking Claude to "sync slack to asana".

---

## Prerequisites

This skill runs entirely through MCP servers configured in Claude Code — no API tokens or `.env` file needed. You need two MCPs connected: **Slack** and **Asana**.

---

## 1. Connect Slack MCP

Slack provides a hosted MCP server with OAuth — no tokens to manage.

### Add to Claude Code

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

### Authenticate

The first time the skill runs, Claude Code will open a browser window to complete the Slack OAuth flow. You'll need to:
1. Sign in to your Slack workspace
2. Approve the requested permissions (see scopes below)

**Required Slack scopes:**
- `search:read` — search messages across channels
- `channels:history` — read public channel messages
- `groups:history` — read private channel messages
- `users:read` + `users:read.email` — resolve user profiles
- `reactions:read` — detect emoji reactions
- `files:read` — detect file attachments

> If your workspace requires admin approval for new app connections, ask your Slack admin to approve the Slack MCP app.

---

## 2. Connect Asana MCP

Asana provides an official V2 MCP server. Use the V2 URL — the V1 beta is deprecated and shuts down May 2026.

### Add to Claude Code

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

### Authenticate

1. Go to [Asana Developer Console](https://app.asana.com/0/my-apps)
2. Create a new app — select **MCP app** as the app type
3. Note your `Client ID` and `Client Secret`
4. Set redirect URL to: `http://localhost:8080/callback`
5. Run the add command with your credentials:

```bash
claude mcp add --transport http \
  --client-id YOUR_CLIENT_ID \
  --client-secret YOUR_CLIENT_SECRET \
  --callback-port 8080 \
  asana https://mcp.asana.com/v2/mcp
```

The first run will open a browser for OAuth. Credentials are stored in your system keychain — no tokens in config files.

**Alternative (token-based, simpler):**

If you prefer a Personal Access Token instead of OAuth:

```bash
npm install -g @roychri/mcp-server-asana
```

```json
{
  "mcpServers": {
    "asana": {
      "command": "npx",
      "args": ["-y", "@roychri/mcp-server-asana"],
      "env": {
        "ASANA_ACCESS_TOKEN": "your-pat-here"
      }
    }
  }
}
```

Get your PAT at: `https://app.asana.com/0/my-apps` → Personal Access Token.

---

## 3. Verify Both MCPs Are Connected

In Claude Code, run:

```
/mcp
```

You should see both `slack` and `asana` listed as connected. If either shows as disconnected, re-run the `claude mcp add` command for that server.

---

## 4. Configure Runtime Settings

The only config you need to set is in `.env` — identity and runtime preferences (no API tokens).

```bash
cp .env.example .env
```

Fill in:

```
USER_NAME=Your Name
ASANA_PROJECT_ID=1213494960962920
LOOKBACK_DAYS=3
STALE_TICKET_SKIP=Lotus
DASHBOARD_OUTPUT_PATH=/absolute/path/to/dashboard_data.json
```

`SLACK_USER_ID` and `ASANA_USER_ID` can be left blank — the skill resolves them at runtime from your MCP connections.

---

## 5. Run the Skill

In Claude Code:

```
/customer-slack-to-asana
```

Or just say: **"sync slack to asana"** or **"scan customer channels"** — the skill triggers automatically from those phrases.

---

## Project Structure

```
SKILL.md          — full skill definition (entry point)
CLAUDE.md         — project rules and context
.env              — runtime settings (no secrets needed)
.env.example      — settings template
workflows/        — detailed per-step reference (01–10)
output/{date}/    — daily run outputs
README.md         — this file
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Slack MCP not finding private channels | Ensure `groups:history` scope is approved in OAuth |
| Asana MCP returns no tasks | Confirm `ASANA_PROJECT_ID` is set correctly in `.env` |
| OAuth flow doesn't open | Run `claude mcp remove slack` then re-add |
| V1 Asana URL errors | Update config URL to `https://mcp.asana.com/v2/mcp` |
| Skill doesn't trigger | Check `/mcp` — both servers must show as connected |
