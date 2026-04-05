# Step 7: Write One Consolidated Comment per Asana Ticket

## Goal
Post exactly one new comment per Asana ticket per run, containing all open and resolved action items in a skimmable format. Never duplicate a comment already present.

## Inputs
- `output/{date}/asana/channel_ticket_map.json` (from Step 6)
- `output/{date}/asana/tickets.json` — existing comments for dedup check
- Asana PAT from `.env`

## Actions

### 1. Check for Existing Sync Comment
- Compare the comment to be written against `existing_comment_texts` in `tickets.json`
- If the same set of action items was already posted (match on item descriptions + statuses), skip this ticket
- Only post if there is at least one new or newly-resolved item since last run

### 2. Format the Comment
Build a skimmable plain-text comment:

```
🔄 Slack Sync — {date}

Open Issues
• [open] Client cannot map 'product_category' field → [screenshot] https://teamairops.slack.com/archives/...
• [open] Webhook not triggering on publish event

Resolved
• [resolved ✅] Auth token expiry issue — confirmed working by client

Channels: #ext-acmecorp, #airops-acmecorp
```

**Formatting rules:**
- Section headers: `Open Issues` and `Resolved` (omit a section if empty)
- Each item: `• [status] {description}`
- If `has_attachment: true`, append `→ [attachment] {attachment_slack_url}` on the same line
- No markdown (Asana comments render plain text)
- Max ~20 items total; if more, group by sub-topic with a brief header
- End with `Channels: #channel1, #channel2` listing all source channels

### 3. Post the Comment
Call:
```
POST /tasks/{task_gid}/stories
Content-Type: application/json

{
  "data": {
    "text": "{formatted_comment}"
  }
}
```

### 4. Log the Result
- On success: record `{ ticket_gid, comment_gid, posted_at }` in `output/{date}/asana/posted_comments.json`
- On failure: record error in `output/{date}/asana/errors.json` and continue to next ticket

## Rules (from CLAUDE.md)
- ONE comment per ticket per run — no exceptions
- Never post if all action items are already reflected in an existing comment
- Distill threads to essentials — no raw message dumps
- Always include Slack URL for any item with a file attachment
