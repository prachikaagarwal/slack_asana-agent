# Step 4: Handle File Attachments

## Goal
Ensure every action item that originated from a message containing a file attachment (screenshot, PDF, doc, etc.) carries a properly formatted Slack deep-link URL so it can be referenced directly in the Asana comment.

## Inputs
- Raw message data from Step 1 (before consolidation)
- `output/{date}/slack/consolidated_{channel_name}.json`

## Actions

### 1. Detect Attachments in Messages
When processing messages in Step 1, flag a message if:
- `message.files` array is non-empty
- `message.attachments` array contains items with `mimetype` (image/*, application/pdf, etc.)
- Message text contains a Slack file share link (`https://files.slack.com/...`)

### 2. Construct the Slack Deep-Link URL

**For top-level channel messages:**
```
https://teamairops.slack.com/archives/{channel_id}/p{message_ts_no_dot}
```

**For thread replies:**
```
https://teamairops.slack.com/archives/{channel_id}/p{message_ts_no_dot}?thread_ts={parent_ts}&cid={channel_id}
```

**Timestamp formatting:**
- Raw Slack timestamp: `1712345678.123456`
- `message_ts_no_dot`: remove the `.` → `1712345678123456`
- `parent_ts`: the `ts` of the root message in the thread (keep the `.`)

### 3. Attach URL to Action Item
- Set `has_attachment: true` on the action item
- Set `attachment_slack_url` to the constructed URL
- In the Asana comment output (Step 6), this URL must always be included inline with the action item

### 4. Multiple Attachments in One Thread
- If multiple messages in the same thread have attachments, include all URLs as a list under `attachment_slack_urls: []`

## Output
Each action item with an attachment will have:
```json
{
  "has_attachment": true,
  "attachment_slack_url": "https://teamairops.slack.com/archives/C12345/p1712345678123456?thread_ts=1712345600.000000&cid=C12345"
}
```
