# Step 5: Fetch Asana Tickets Assigned to User

## Goal
Pull all open Asana tickets assigned to `$ASANA_USER_ID` in project `$ASANA_PROJECT_ID` to use as the target for Slack issue mapping and comment writing.

**Authentication:** via the Asana MCP configured in Claude Code. No API tokens needed.

## Inputs
- `$ASANA_USER_ID` — from `.env`, or resolved at runtime via the Asana MCP
- `$ASANA_PROJECT_ID` — from `.env`

## Actions

### 1. Resolve User's Asana ID (if blank)
If `$ASANA_USER_ID` is not set, use the Asana MCP to look up the current authenticated user and store their GID.

### 2. Fetch Incomplete Tasks in Project
Use the Asana MCP to fetch all incomplete tasks in project `$ASANA_PROJECT_ID` assigned to `$ASANA_USER_ID`.

### 3. Extract Company Name from Ticket Title
- Each ticket title is expected to contain a `{companyname}` identifier
- Normalize: lowercase, strip punctuation/spaces → slugified form
- Example: `"Acme Corp - CMS Integration"` → `acmecorp`
- Store as `ticket.company_slug` for matching in Step 6

### 4. Fetch Existing Comments (to avoid duplicates)
For each ticket, use the Asana MCP to fetch existing stories/comments. Store:
- Most recent comment `created_at` as `last_comment_at`
- Full comment texts to check for `<!-- customer-slack-sync -->` and `<!-- customer-slack-stale-nudge -->` markers in Steps 7 and 9

## Output
Write to `output/{date}/asana/tickets.json`:
```json
[
  {
    "gid": "9876543210",
    "name": "Acme Corp - CMS Integration",
    "company_slug": "acmecorp",
    "completed": false,
    "last_comment_at": "2026-04-01T12:00:00Z",
    "existing_comment_texts": ["Previous sync comment..."]
  }
]
```
