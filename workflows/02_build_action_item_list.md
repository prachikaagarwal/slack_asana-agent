# Step 2: Build Consolidated Action Item List per Channel

## Goal
Merge all extracted messages and thread replies from Step 1 into a single, deduplicated list of `{action_items}` per Slack `{channel}`.

## Inputs
- `output/{date}/slack/{channel_name}.json` files from Step 1
- Previous run outputs to detect already-known issues

## Actions

### 1. Load Extracted Data
- Read all `output/{date}/slack/*.json` files produced in Step 1

### 2. Deduplicate Action Items
- Compare against action items from previous run outputs in `output/{previous_date}/slack/`
- Skip any action item whose `description` closely matches an already-known item (use semantic similarity or exact match on keywords + channel)
- Assign stable IDs to carry across runs

### 3. Group by Channel
- Each channel maps to one company (`company_name` from channel name)
- Group all action items under their respective channel/company

### 4. Classify Each Action Item
Assign one of:
- `open` — unresolved, no positive signal
- `resolved` — see Step 3 for resolution detection (pass-through at this stage, mark pending)
- `in_progress` — explicitly mentioned as being worked on

### 5. Sort & Prioritize
- Oldest unresolved items first
- Items with file attachments flagged at top of list
- Items where client is waiting on Airops team flagged

## Output
Write to `output/{date}/slack/consolidated_{channel_name}.json`:
```json
{
  "company_name": "acmecorp",
  "channel_id": "C12345",
  "channel_name": "ext-acmecorp",
  "action_items": [
    {
      "id": "ai_001",
      "description": "Client cannot map 'product_category' field in CMS connector",
      "keywords": ["field mapping", "product_category"],
      "status": "open",
      "slack_url": "https://teamairops.slack.com/archives/C12345/p1712345678123456",
      "has_attachment": true,
      "first_seen": "2026-04-02T10:00:00Z"
    }
  ]
}
```
