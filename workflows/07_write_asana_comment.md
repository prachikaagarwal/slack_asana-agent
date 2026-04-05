# Step 7: Write One Consolidated Comment per Asana Ticket

## Goal
Post exactly one new comment per Asana ticket per run, containing all open and resolved action items in a skimmable format. Never duplicate a comment already present.

**Authentication:** via the Asana MCP configured in Claude Code. No API tokens needed.

## Inputs
- `output/{date}/asana/channel_ticket_map.json` (from Step 6)
- `output/{date}/asana/tickets.json` — existing comments for dedup check

## Actions

### 1. Check for Existing Sync Comment
- Scan `existing_comment_texts` in `tickets.json` for the `<!-- customer-slack-sync -->` marker
- If the same set of action items is already reflected in a prior comment → skip this ticket
- Only post if there is at least one new or newly-resolved item since last run

### 2. Format the Comment
```
<!-- customer-slack-sync -->
📋 Integration Issues Summary — [Customer Name]
Last synced: [YYYY-MM-DD HH:MM]

---

**Issue 1: [Short title]**
📅 Detected: [YYYY-MM-DD]
🔴 Status: Open

**Pain Point:** [1-2 sentences]
**Current Status:** [1-2 sentences]
**Ideal Status:** [1-2 sentences]

---

**Issue 2: [Short title]**
📅 Detected: [YYYY-MM-DD]
✅ Status: Resolved

**Pain Point:** [1-2 sentences]
**Current Status:** Confirmed resolved in Slack thread.
**Ideal Status:** [1-2 sentences]

---

_Synced from Slack by the customer-slack-to-asana skill. Next sync: tomorrow 9am._
```

For any issue with a file attachment, append on the same line:
`→ [attachment] https://app.slack.com/archives/{channel_id}/p{ts_no_dot}?thread_ts={parent_ts}&cid={channel_id}`

### 3. Post the Comment
Use the Asana MCP to add a story/comment to the task with the formatted text above.

### 4. Log the Result
- On success: record `{ ticket_gid, comment_gid, posted_at }` in `output/{date}/asana/posted_comments.json`
- On failure: record error in `output/{date}/asana/errors.json` and continue to next ticket

## Rules
- ONE comment per ticket per run — no exceptions
- Never post if all action items are already reflected in an existing comment
- Distill threads to essentials — no raw message dumps
- Always include Slack URL for any item with a file attachment
