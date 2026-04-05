# Step 1: Extract Keywords & Action Items from Slack

## Goal
Pull all messages and threads from relevant Slack channels for the last `$LOOKBACK_DAYS` days and extract structured keywords and action items.

**Authentication:** via the Slack MCP configured in Claude Code. No tokens needed.

## Inputs
- `$USER_NAME` / `$SLACK_USER_ID` — to identify the user's channels
- `$LOOKBACK_DAYS` — scan window (default: 3 days, or since last run)
- `$CHANNEL_PREFIXES` — channel name prefixes to filter customer channels

## Actions

### 1. Resolve User's Slack ID
Use the Slack MCP to look up the user by `$USER_NAME` if `$SLACK_USER_ID` is blank. Store the `U...` ID for use throughout the run.

### 2. Find Relevant Channels
Use the Slack MCP to list channels the user belongs to. Filter to names starting with any prefix in `$CHANNEL_PREFIXES` (`ext-`, `airops-`, `ext-airops-`). Store as `{ channel_id, channel_name, company_name }` where `company_name` is parsed from the suffix after the prefix.

### 3. Compute Lookback Timestamp
Check `output/` for the most recent run date. Use `max(today - $LOOKBACK_DAYS, last_run_date)` as the oldest boundary for message fetching.

### 4. Fetch Messages and Threads
For each channel, use the Slack MCP to fetch messages since the lookback timestamp. For any message with thread replies, fetch the full thread. Threads are critical — context and resolutions often appear there.

### 5. Extract Keywords & Action Items
For each message and thread reply:
- **Keywords:** CMS platform names, integration terms, error messages, field names
- **Action items:** tasks, blockers, requests, follow-ups, unanswered questions
- **Flag attachments:** any message containing files (images, PDFs, docs) — capture `channel_id`, `message_ts`, and `thread_ts` for URL construction

### 6. Build Slack Deep-Link URLs
For messages with attachments, construct the direct Slack URL:

**Top-level message:**
```
https://app.slack.com/archives/{channel_id}/p{message_ts_no_dot}
```

**Thread reply:**
```
https://app.slack.com/archives/{channel_id}/p{message_ts_no_dot}?thread_ts={parent_ts}&cid={channel_id}
```

`message_ts_no_dot` = timestamp with `.` removed (e.g. `1712345678.123456` → `1712345678123456`)

## Output
Write to `output/{date}/slack/{channel_name}.json`:
```json
{
  "channel_id": "C12345",
  "channel_name": "ext-acmecorp",
  "company_name": "acmecorp",
  "extracted_at": "YYYY-MM-DDTHH:MM:SSZ",
  "action_items": [
    {
      "id": "ai_001",
      "description": "...",
      "keywords": ["CMS", "field mapping"],
      "status": "open",
      "source_message_ts": "1712345678.123456",
      "thread_ts": "1712345600.000000",
      "slack_url": "https://app.slack.com/archives/C12345/p1712345678123456",
      "has_attachment": false
    }
  ]
}
```
