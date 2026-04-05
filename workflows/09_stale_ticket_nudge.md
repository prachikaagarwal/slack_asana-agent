# Step 9: Stale Ticket Nudge

## Goal
After the main sync, check all open Asana tickets assigned to `$ASANA_USER_ID` for staleness and post a single status nudge comment if a ticket has gone quiet.

**Authentication:** via the Slack MCP and Asana MCP. No API tokens needed.

## Inputs
- All open Asana tickets assigned to `$ASANA_USER_ID` in project `$ASANA_PROJECT_ID`
- `output/{date}/asana/tickets.json` — existing comments per ticket
- Slack channel data from Step 3

## Staleness Conditions
A ticket is stale if **BOTH** are true:
1. No comment added in 3+ days
2. No meaningful Slack update from `$SLACK_USER_ID` in the matching customer channel in the last `$LOOKBACK_DAYS` days

**What counts as a Slack update:** A message from `$SLACK_USER_ID` OR a message from a customer/SA the user should be aware of (client feedback, blocker resolution, new action items).

**Does NOT count:** Bot joins, AirOps Customers bot messages, purely administrative messages.

## Actions

### 1. Load All Open Tickets
Re-use the ticket list from Step 5. Filter to only incomplete tickets assigned to `$ASANA_USER_ID`.

### 2. Check Each Ticket for Staleness
- If last comment was < 3 days ago → **not stale**, skip
- If last comment was 3+ days ago → check matching Slack channel for activity via Slack MCP
- If Slack also has no meaningful update → **stale**

### 3. Check for Existing Nudge
Scan `existing_comment_texts` for `<!-- customer-slack-stale-nudge -->`. If one was added within the last 3 days → skip.

### 4. Skip Rules
- Skip any ticket whose name matches an entry in `$STALE_TICKET_SKIP` (set in `.env`)
- Skip any ticket where `$ASANA_USER_ID` commented in Asana within the last 3 days

### 5. Post the Nudge Comment
Use the Asana MCP to add a story/comment to the task:

```
<!-- customer-slack-stale-nudge -->
📋 Status Update — [Customer Name]
Last synced: [YYYY-MM-DD]

**Last known status (from Slack + Asana):**
[1-3 sentences: what's done, what's blocked, what's waiting on whom]

**Blocker:** [What's preventing progress, if any]
**Next step:** [The single most important action needed]

_Auto-nudge: No Asana comment in [N]+ days. Synced from Slack by customer-slack-to-asana skill._
```

Keep it skimmable — distill from Slack threads + existing Asana comments. Do not paste raw messages.

### 6. Log Results
Record each nudge posted in `output/{date}/asana/stale_nudges.json`:
```json
[
  {
    "ticket_gid": "9876543210",
    "ticket_name": "Acme Corp - CMS Integration",
    "nudge_comment_gid": "1122334455",
    "days_since_last_comment": 5,
    "posted_at": "YYYY-MM-DDTHH:MM:SSZ"
  }
]
```

## Output
Count of nudges added, included in Step 10 final report.
