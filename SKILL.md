---
name: customer-slack-to-asana
description: >
  Daily sync of customer Slack feedback to Asana tickets. Scans #ext-[company], #airops-[company], and #ext-airops-[company] channels for the last 3 days, extracts integration issues per client (pain point, current status, ideal status), detects resolution from the user's replies or emoji reactions, and writes ONE consolidated comment per Asana ticket. Also saves dashboard_data.json and posts stale ticket nudges. Trigger when the user asks to "scan customer channels", "sync slack to asana", "check what customers need", "what did customers say this week", or when the scheduled 9am daily trigger fires.
---

# Customer Slack → Asana Daily Sync

## Before You Begin

This skill uses the **Slack MCP** and **Asana MCP** configured in Claude Code — no API tokens needed. If either MCP is not connected, stop and tell the user to follow the setup steps in `README.md`.

Authentication is handled by the **Slack MCP** and **Asana MCP** — no API keys needed. If either MCP is disconnected, stop and direct the user to `README.md`.

Read runtime settings from `.env` if present (copy from `.env.example`): `USER_NAME`, `SLACK_USER_ID`, `ASANA_USER_ID`, `ASANA_PROJECT_ID`, `LOOKBACK_DAYS`, `CHANNEL_PREFIXES`, `STALE_TICKET_SKIP`, `DASHBOARD_OUTPUT_PATH`. All are optional except `ASANA_PROJECT_ID` — missing IDs are resolved at runtime via MCP.

---

## Step 1 — Get User's Slack ID

Read `$SLACK_USER_ID` from `.env`. If blank, use the Slack MCP to search for the user by `$USER_NAME` and store the returned `U...` ID. You'll need it for @mention detection and resolution checking in every step.

---

## Step 2 — Find All Customer Channels

Run both searches, merge, and deduplicate results:

**A. Channel name search** — use the Slack MCP to search channels for each prefix in `$CHANNEL_PREFIXES`:
- `airops-{company}` → e.g. `#airops-zenbusiness`, `#airops-lawnstarter`
- `ext-{company}` → e.g. `#ext-acmecorp`
- `ext-airops-{company}` → e.g. `#ext-airops-ping`, `#ext-airops-adobe`
- `airops-customer-{company}` → e.g. `#airops-customer-acme`

**B. @mention search** — use the Slack MCP to search messages mentioning `$SLACK_USER_ID` across all channels in the last `$LOOKBACK_DAYS` days.

Skip internal-only channels (alerts, standups, CI/CD, random, bot channels).

**Normalize company name from channel name** — strip the prefix, then capitalize:
- `#airops-zenbusiness` → "ZenBusiness"
- `#ext-airops-ping` → "Ping"
- `#airops-customer-acme` → "Acme"
- `#airops-lawnstarter` → "LawnStarter"

---

## Step 3 — Read Messages and Threads

For each customer channel, use the Slack MCP to fetch messages from the last `$LOOKBACK_DAYS` days.

Flag a message as relevant if ANY of:
1. From a non-AirOps user with feedback, a complaint, a question, or a request
2. Mentions integration keywords: integration, sync, API, webhook, mapping, field, Salesforce, HubSpot, WordPress, CMS, plugin, Contentstack, AEM, connector, publish, draft, schema, field mapping
3. Directly @mentions `$SLACK_USER_ID`

For every relevant message with thread replies, fetch the full thread via the Slack MCP. Threads are critical — resolutions and context often appear there, not in the original message.

---

## Step 4 — Build Issue List Per Customer

For each channel, compile all distinct integration issues from the lookback window.

```
Issue #N
  Short title:    [5-8 words, e.g. "Yoast meta description not syncing"]
  Pain point:     [What is broken or frustrating — be specific]
  Current status: [What is happening right now]
  Ideal status:   [What the customer wants to happen]
  Date found:     [YYYY-MM-DD]
  Resolution:     [Open | Resolved]
```

- Combine messages about the same root cause into one entry
- Separate entries for clearly distinct issues (e.g. broken field mapping vs. webhook timeout vs. feature request)
- Flag any message with a file attachment — you'll need its Slack URL for the Asana comment

**Constructing Slack URLs for attachments:**
- Top-level message: `https://app.slack.com/archives/{channel_id}/p{ts_no_dot}`
- Thread reply: `https://app.slack.com/archives/{channel_id}/p{ts_no_dot}?thread_ts={parent_ts}&cid={channel_id}`
- `ts_no_dot` = timestamp with `.` removed (e.g. `1712345678.123456` → `1712345678123456`)

---

## Step 5 — Detect Resolution

Check both places for resolution signals. Only signals from `$SLACK_USER_ID` count.

**A. Top-level channel messages from the user:**
Look for messages posted by `$SLACK_USER_ID` directly in the channel (not in a thread) that contain phrases like "now resolved", "these are fixed", "all done", "completed", "pushed a fix" and list items with bullet points. Match each listed item against open issues and mark them **Resolved**. One message can batch-resolve multiple issues.

Also extract items flagged as still open or needing follow-up in the same message (e.g. "still working on:", "a few quick notes:") → new Open items.

**B. Thread replies from the user:**
For each issue's thread, mark **Resolved** if `$SLACK_USER_ID` posted a reply containing any of:
`resolved`, `done`, `fixed`, `deployed`, `pushed`, `shipped`, `updated`, `live`, `this is now`, `going out`, `merged`, `working now`

**C. ✅ reaction** on the original message → **Resolved**.

If none of the above → **Open**. Resolution can come from any reply at any time, not just same-day.

---

## Step 6 — Match Asana Tickets

For each customer, find their Asana ticket:

1. Use the Asana MCP to fetch all incomplete tasks in project `$ASANA_PROJECT_ID` assigned to `$ASANA_USER_ID`
2. Match customer name as a case-insensitive substring of the ticket title
3. Prefer incomplete tickets; prefer the most topically specific match
4. For each matched ticket, use the Asana MCP to fetch existing stories/comments — store texts and timestamps for dedup

If no ticket found for a customer → log them as unmatched, skip Asana write for that customer.

---

## Step 7 — Write One Consolidated Asana Comment

**One comment per ticket per run. Never post if nothing has changed.**

Before posting, check existing comment texts for `<!-- customer-slack-sync -->`. If all current issues are already reflected in a prior comment → skip this ticket.

Post using the Asana MCP to add a story/comment to the task.

**Comment format:**
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

Always append the Slack URL for any issue whose source message had a file attachment:
`→ [attachment] {slack_url}`

---

## Step 8 — Save Dashboard JSON

Write `dashboard_data.json` to `$DASHBOARD_OUTPUT_PATH`:

```json
{
  "last_synced": "YYYY-MM-DDTHH:MM:SS",
  "customers": [
    {
      "name": "ZenBusiness",
      "channel": "#airops-zenbusiness",
      "asana_ticket": "[ticket name]",
      "asana_ticket_id": "[ticket gid]",
      "issues": [
        {
          "title": "Yoast meta description not syncing",
          "pain_point": "...",
          "current_status": "...",
          "ideal_status": "...",
          "date_found": "YYYY-MM-DD",
          "resolution": "Open"
        }
      ]
    }
  ],
  "unmatched_customers": [
    { "name": "Fortinet", "channel": "#ext-airops-ping", "issues": [] }
  ]
}
```

---

## Step 9 — Stale Ticket Nudge

Check all open tickets assigned to `$ASANA_USER_ID` for staleness. A ticket is stale if **both**:
1. No comment added in 3+ days
2. No meaningful Slack update from `$SLACK_USER_ID` in the matching channel in the last `$LOOKBACK_DAYS` days

Bot joins, AirOps Customers bot messages, and administrative messages do NOT count as updates.

**Skip a ticket if any of:**
- Its name matches any entry in `$STALE_TICKET_SKIP`
- `$ASANA_USER_ID` commented on it in the last 3 days
- A `<!-- customer-slack-stale-nudge -->` comment exists from the last 3 days

**Post nudge comment:**
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

---

## Step 10 — Report

Output this summary when done:

```
✅ Sync complete — [YYYY-MM-DD HH:MM]

Channels scanned: [N]
Customers with issues: [N]
Total issues found: [N] ([N] open, [N] resolved)

Asana tickets updated:
- [Ticket name] → [Customer]: [N] issues ([N] open, [N] resolved)

Customers without an Asana ticket (create manually):
- [Customer] — [N] issues: [brief description]

Stale ticket nudges added: [N]
- [Ticket name]: [one-line status summary]

Dashboard data saved ✓
```

Omit any section that has nothing to report. If dashboard write failed, say `Dashboard data save FAILED: [error]`.
