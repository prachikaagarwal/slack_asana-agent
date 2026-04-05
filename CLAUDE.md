# Project Context
Slack-Asana sync agent for a CX Integration Engineer. Scans customer Slack channels, extracts CMS integration issues per client, detects resolution, and writes consolidated comments to Asana tickets.

**To run:** invoke the `customer-slack-to-asana` skill. All execution logic lives in `SKILL.md`. Detailed step reference in `workflows/`.

**Authentication:** handled entirely by the Slack MCP and Asana MCP configured in Claude Code. No API keys or tokens anywhere in this project.

# Runtime Config
Non-sensitive runtime settings only. Stored in `.env.example` — copy to `.env` to override defaults.

| Variable | Default | Purpose |
|---|---|---|
| `USER_NAME` | — | Used to resolve Slack user ID at runtime via Slack MCP |
| `SLACK_USER_ID` | resolved at runtime | Slack member ID for @mention detection and resolution signals |
| `ASANA_USER_ID` | resolved at runtime | Asana user GID for filtering assigned tickets |
| `ASANA_PROJECT_ID` | `1213494960962920` | Target Asana project |
| `LOOKBACK_DAYS` | `3` | Days back to scan Slack |
| `CHANNEL_PREFIXES` | `ext-,airops-,ext-airops-` | Customer channel prefixes |
| `STALE_TICKET_SKIP` | — | Ticket names to skip for stale nudges |
| `DASHBOARD_OUTPUT_PATH` | — | Where to write `dashboard_data.json` |

# Output Structure
```
output/{date}/slack/          — extracted issues per channel
output/{date}/asana/          — ticket maps, posted comments, errors, stale nudges
$DASHBOARD_OUTPUT_PATH        — dashboard_data.json
```

# Rules
- Scan back to last run timestamp, minimum `$LOOKBACK_DAYS` days
- One comment per Asana ticket per run — check for `<!-- customer-slack-sync -->` before posting
- Only `$SLACK_USER_ID`'s replies and reactions count as resolution signals
- Always include Slack deep-link URL for any message with a file attachment
- Stale nudge dedup: check for `<!-- customer-slack-stale-nudge -->` within last 3 days before posting
- Asana comments must be skimmable — summarize, never paste raw Slack messages

# Project Structure
```
SKILL.md          — executable skill (entry point for Claude Code)
CLAUDE.md         — this file: project rules and context
.env.example      — runtime settings template (no secrets)
.env              — local overrides (never commit)
workflows/        — detailed per-step reference docs (01–10)
output/{date}/    — daily run outputs
README.md         — MCP setup instructions
```
