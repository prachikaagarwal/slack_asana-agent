# Step 1: Extract Keywords & Action Items from Slack

## Goal
Pull all messages and threads from relevant Slack channels for the last 3 days and extract structured `{keywords}` and `{action_items}`.

## Inputs
- `{user_ID_slack}` — used to identify which channels the user is a member of
- `{lookback_window}` — last 3 days, or since last run (whichever is more recent)
- Channel filter: channels starting with `#ext-{companyname}` or `#airops-{companyname}`

## Actions

### 1. Authenticate with Slack
- Use the Slack OAuth token from `.env`
- Scope required: `search:read`, `channels:history`, `groups:history`, `im:history`, `users:read`

### 2. List Relevant Channels
- Call `conversations.list` to get all channels the user belongs to
- Filter: channel names starting with `ext-` or `airops-`
- Store as a list of `{ channel_id, channel_name, company_name }` where `company_name` is parsed from the channel name suffix

### 3. Compute Lookback Timestamp
- Check `output/{last_run_date}/` to find the most recent run
- Use `max(3 days ago, last_run_timestamp)` as the `oldest` param for Slack API calls

### 4. Fetch Messages per Channel
- For each channel, call `conversations.history` with `oldest={lookback_timestamp}`
- For each message that has `reply_count > 0`, call `conversations.replies` to fetch the full thread

### 5. Extract Keywords & Action Items
For each message and thread reply:
- Identify `{keywords}`: CMS platform names, integration terms, error messages, field names
- Identify `{action_items}`: tasks, blockers, requests, follow-ups, unanswered questions
- Flag messages containing file attachments (images, PDFs, docs) — capture `channel_id`, `message_ts`, `thread_ts` for URL construction

### 6. Build Slack Message URL
For thread replies with attachments, construct:
```
https://teamairops.slack.com/archives/{channel_id}/p{message_ts_no_dot}?thread_ts={parent_ts}&cid={channel_id}
```
- `message_ts_no_dot` = timestamp with `.` removed (e.g., `1712345678.123456` → `1712345678123456`)

## Output
Write to `output/{date}/slack/{channel_name}.json`:
```json
{
  "channel_id": "C12345",
  "channel_name": "ext-acmecorp",
  "company_name": "acmecorp",
  "extracted_at": "2026-04-04T00:00:00Z",
  "action_items": [
    {
      "id": "ai_001",
      "description": "...",
      "keywords": ["CMS", "field mapping"],
      "status": "open",
      "source_message_ts": "1712345678.123456",
      "thread_ts": "1712345600.000000",
      "slack_url": "https://teamairops.slack.com/...",
      "has_attachment": false
    }
  ]
}
```
