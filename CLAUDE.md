# Project Context
This is a Slack-Asana Agent that scans Slack customer channels for the last 3 days, builds a complete list of integration issues per client (pain point, current status, ideal status), detects resolution from `$USER_NAME`'s replies or emoji reactions, writes ONE consolidated comment per Asana ticket covering all open and resolved issues, and saves a `dashboard_data.json` for the action item HTML dashboard.

**Skill trigger:** Use this whenever the user asks to "scan customer channels", "sync slack feedback to asana", "check what customers need", "what did customers say this week", or when the scheduled 9am daily trigger fires. Trigger proactively if the user mentions wanting a summary of customer integration requests or wants to know what they should be working on.

# About Me
> All identity and credential values are stored in `.env` (or `.env`). Never hardcode them here.

Role: CX Integration Engineer — works with customers to solve CMS integration issues, ensures clients can connect Airops with their CMS platform, and helps Solutions Architects understand existing fields in clients' CMS structure.

| Config variable | Where to set |
|---|---|
| `$USER_NAME` | `.env` |
| `$USER_EMAIL` | `.env` |
| `$SLACK_USER_ID` | `.env` (or resolved at runtime) |
| `$ASANA_USER_ID` | `.env` |
| `$ASANA_PROJECT_ID` | `.env` (default: `1213494960962920`) |
| `$DASHBOARD_OUTPUT_PATH` | `.env` |

# Output Paths
- `output/{date}/slack/` — extracted action items per channel
- `output/{date}/asana/` — ticket maps, posted comments, errors
- `$DASHBOARD_OUTPUT_PATH/dashboard_data.json` — dashboard JSON (path set in `.env`)

# Project Structure
- `workflows/` — workflow instruction files (one per step)
- `.env` — all user identity, API tokens, runtime settings (never commit)
- `.env.example` — machine-readable credential template
- `output/{date}/slack/` — daily slack extraction
- `output/{date}/asana/` — daily asana updates

# Rules
- Always look back to last time since your last run (minimum `$LOOKBACK_DAYS` days)
- Don't add comments already added in Asana — check for `<!-- customer-slack-sync -->` marker
- Never write more than one new sync comment per Asana ticket per run — consolidate all issues into one
- Asana comments should be skimmable — distill Slack threads down to the essentials
- Resolution is detected from **`$USER_NAME`'s** replies or emoji reactions only
- File attachments → always include Slack link. For thread messages use: `https://$SLACK_WORKSPACE_URL/archives/{channel_id}/p{message_ts_no_dot}?thread_ts={parent_ts}&cid={channel_id}`
- Integration focus: field mappings, CMS plugins, API webhooks, and content sync issues are core domain — prioritize these
- Stale nudge dedup: use `<!-- customer-slack-stale-nudge -->` marker; only add a new one if last nudge was 3+ days ago

# Steps

## Step 1 — Get User's Slack User ID
- Read `$SLACK_USER_ID` from `.env` / `.env`
- If blank, resolve at runtime: use `slack_search_users` with query `$USER_NAME` to get the `U...` ID
- Store it for @mention detection and resolution checking throughout the run

## Step 2 — Find All Customer Channels
Run **both** searches and merge/deduplicate:

**Search A — Channel name patterns** (prefixes from `$CHANNEL_PREFIXES`):
- `airops-{companyname}` (e.g. `#airops-zenbusiness`, `#airops-lawnstarter`)
- `ext-{companyname}` (e.g. `#ext-acmecorp`)
- `ext-airops-{companyname}` (e.g. `#ext-airops-ping`, `#ext-airops-adobe`) — shared external channels with a different prefix; don't miss these
- `airops-customer-{companyname}` (e.g. `#airops-customer-acme`)

**Search B — @user mentions:** Use `slack_search_public_and_private` with `$SLACK_USER_ID` to find every channel where the user was mentioned in the last `$LOOKBACK_DAYS` days.

Skip purely internal channels (alerts, standups, CI/CD, random).

**Customer name normalization:**
- Strip the channel prefix, then capitalize appropriately
- `#airops-zenbusiness` → "ZenBusiness"
- `#airops-customer-acme` → "Acme"
- `#ext-cust-salesforce` → "Salesforce"
- `#airops-lawnstarter` → "LawnStarter"

## Step 3 — Read Messages and Threads
For each customer channel, use `slack_read_channel` to get messages from the last `$LOOKBACK_DAYS` days.

Flag a message as relevant if ANY of:
1. It's from a customer (not AirOps staff) with feedback, complaint, question, or request
2. It mentions integrations — keywords: integration, sync, API, webhook, mapping, field, Salesforce, HubSpot, WordPress, CMS, plugin, Contentstack, AEM, etc.
3. It directly @mentions `$SLACK_USER_ID`

For every relevant message with thread replies, read the full thread with `slack_read_thread`. Thread replies are critical for resolution detection.

## Step 4 — Build a Complete Issue List Per Customer
For each channel, compile all distinct integration issues found in the lookback window. Each issue:

```
Issue #N:
  Short title:    [5-8 word description, e.g. "Yoast meta description not syncing"]
  Pain Point:     [What is broken or frustrating — be specific]
  Current Status: [What is happening right now]
  Ideal Status:   [What the customer wants]
  Date found:     [YYYY-MM-DD]
  Resolution:     [Open | Resolved]
```

Combine messages about the same root issue into one entry. Separate entries for clearly distinct issues.

## Step 5 — Determine Resolution Status Per Issue
Check **both** places:

**A. Top-level messages from `$USER_NAME` in the channel:**
Messages like "hey team! the following issues are now resolved: • Table formatting • FAQ formatting…" — match listed items against open issues and mark them Resolved. Also extract any items flagged as still open/needing follow-up.

**B. Thread replies from `$USER_NAME`:**
Mark Resolved if `$SLACK_USER_ID` posted a reply containing: "resolved", "done", "fixed", "deployed", "pushed", "shipped", "updated", "live", "this is now", "going out", "merged", "working now" — OR if a ✅ reaction was added to the original message.

Resolution can come from any reply at any time — not just same-day. If no signals → Open.

## Step 6 — Find the Matching Asana Ticket
For each customer, find their Asana ticket:
1. Use `get_my_tasks` or `search_objects` on project `$ASANA_PROJECT_ID` filtered by assignee `$ASANA_USER_ID`
2. Match by customer name as a case-insensitive substring of the ticket title
3. Prefer incomplete tickets; prefer the most topically specific match

If no ticket is found, note in summary — don't write to Asana for that customer.

## Step 7 — Write One Consolidated Comment Per Ticket
**One comment per Asana ticket per run.**

First check for existing `<!-- customer-slack-sync -->` comment — note it, then post a new comment that supersedes it with the freshest full picture.

**Comment format:**
```
<!-- customer-slack-sync -->
📋 Integration Issues Summary — [Customer Name]
Last synced: [YYYY-MM-DD HH:MM]

---

**Issue 1: [Short title]**
📅 Detected: [date]
🔴 Status: Open

**Pain Point:** [description]
**Current Status:** [description]
**Ideal Status:** [description]

---

**Issue 2: [Short title]**
📅 Detected: [date]
✅ Status: Resolved

**Pain Point:** [description]
**Current Status:** Confirmed resolved in Slack thread.
**Ideal Status:** [description]

---

_Synced from Slack by the customer-slack-to-asana skill. Next sync: tomorrow 9am._
```

Keep Pain Point / Current Status / Ideal Status to 1–2 sentences. Always include Slack URL for any message with a file attachment.

## Step 8 — Save Dashboard Data
After processing all customers, write `dashboard_data.json` to `$DASHBOARD_OUTPUT_PATH`:

```json
{
  "last_synced": "YYYY-MM-DDTHH:MM:SS",
  "customers": [
    {
      "name": "ZenBusiness",
      "channel": "#airops-zenbusiness",
      "asana_ticket": "ZenBusiness & Wordpress",
      "asana_ticket_id": "{ticket_gid}",
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

## Step 9 — Stale Ticket Nudge
Check **all open Asana tickets assigned to `$ASANA_USER_ID`** for staleness. A ticket is stale if BOTH:
1. No comment added in 3+ days
2. No meaningful Slack update from `$SLACK_USER_ID` in the matching customer channel in the last `$LOOKBACK_DAYS` days

Bot joins, AirOps Customers bot messages, and purely administrative messages do NOT count as updates.

For each stale ticket, add ONE comment. Skip if:
- A `<!-- customer-slack-stale-nudge -->` was added in the last 3 days
- Ticket name matches any entry in `$STALE_TICKET_SKIP`
- `$ASANA_USER_ID` (the user) commented on the ticket within the last 3 days

**Comment format:**
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

## Step 10 — Final Report
```
✅ Sync complete — [today's date HH:MM]

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
