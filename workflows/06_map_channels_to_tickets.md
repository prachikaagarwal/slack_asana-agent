# Step 6: Map Slack Channels to Asana Tickets

## Goal
Create a mapping between each Slack channel and its corresponding Asana ticket by matching the `{companyname}` slug found in both.

## Inputs
- `output/{date}/slack/consolidated_{channel_name}.json` (from Steps 2–4)
- `output/{date}/asana/tickets.json` (from Step 5)

## Actions

### 1. Normalize Names for Matching
Apply the same normalization to both sides:
- Lowercase
- Remove spaces, hyphens, underscores, punctuation
- Examples:
  - Channel `ext-acme-corp` → `acmecorp`
  - Ticket `Acme Corp - CMS Setup` → `acmecorp`

### 2. Build the Channel-to-Ticket Map
For each Slack channel:
- Extract `company_slug` from channel name (strip `ext-` or `airops-` prefix, then normalize)
- Find the Asana ticket whose `company_slug` matches
- If multiple channels match one ticket (e.g., both `ext-acmecorp` and `airops-acmecorp`), merge their action item lists

### 3. Handle No-Match Cases
- Channel has no matching Asana ticket → log to `output/{date}/asana/unmatched_channels.json`, skip comment creation
- Asana ticket has no matching channel → log to `output/{date}/asana/unmatched_tickets.json`, no action needed

### 4. Merge Multi-Channel Action Items
If both `ext-{company}` and `airops-{company}` exist:
- Combine action item lists, deduplicate by description similarity
- Preserve source channel for each item

## Output
Write to `output/{date}/asana/channel_ticket_map.json`:
```json
[
  {
    "ticket_gid": "9876543210",
    "ticket_name": "Acme Corp - CMS Integration",
    "company_slug": "acmecorp",
    "channels": ["ext-acmecorp", "airops-acmecorp"],
    "action_items": [
      {
        "id": "ai_001",
        "description": "...",
        "status": "open",
        "slack_url": "...",
        "has_attachment": false
      }
    ]
  }
]
```
