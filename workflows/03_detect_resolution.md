# Step 3: Detect Resolution from User's Replies & Reactions

## Goal
Mark action items as `resolved` only when `$USER_NAME` (identified by `$SLACK_USER_ID`) posts a resolution signal in a thread reply or top-level channel message, or when a ✅ reaction is added to the original message.

**Authentication:** via the Slack MCP. No tokens needed.

## Inputs
- `output/{date}/slack/consolidated_{channel_name}.json` from Steps 1/2
- `$SLACK_USER_ID` (from `.env`, or resolved in Step 1 via Slack MCP)

## Actions

### 1. Check Top-Level Channel Messages from the User (Batch Resolution)
Use the Slack MCP to scan for messages posted by `$SLACK_USER_ID` directly in the channel (not in a thread) that:
- Contain phrases: "now resolved", "these are fixed", "all done", "completed", "pushed a fix"
- List items with bullet points

When found, match each listed item against open issues and mark them **Resolved**. One message can batch-resolve multiple issues.

Also extract items flagged as still open or needing follow-up in the same message → new Open action items.

### 2. Check Thread Replies from the User
For each issue's source thread, read all replies via the Slack MCP. Mark as **Resolved** if `$SLACK_USER_ID` posted a reply containing:
- "resolved", "done", "fixed", "deployed", "pushed", "shipped", "updated", "live", "this is now", "going out", "merged", "working now"

Resolution can come from any reply at any time — not just same-day.

### 3. Check for ✅ Reaction
Use the Slack MCP to fetch reactions on the original message. If a ✅ (`:white_check_mark:`) reaction is present → mark as **Resolved**.

### 4. Default
If none of the above signals → status remains **Open**.

### 5. Update Action Item Status
- Set `status`: `"open"` → `"resolved"`
- Add:
  - `resolved_by`: `$SLACK_USER_ID`
  - `resolved_at`: timestamp of the resolution signal
  - `resolution_source`: `"top_level_message"`, `"thread_reply"`, or `"reaction"`

## Output
Update `output/{date}/slack/consolidated_{channel_name}.json` in place with resolved statuses.

## Important Rule
Only signals from `$SLACK_USER_ID` count for resolution detection. Other team members' responses do not mark items resolved.
