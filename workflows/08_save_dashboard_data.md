# Step 8: Save Dashboard Data

## Goal
After all customers are processed, write a `dashboard_data.json` file that powers the HTML action item dashboard.

Output path is read from `$DASHBOARD_OUTPUT_PATH` in `.env` / `.env`.

## Inputs
- `output/{date}/asana/channel_ticket_map.json` (from Step 6) — all processed customers and their issues
- `output/{date}/asana/unmatched_channels.json` — customers with no Asana ticket

## Actions

### 1. Build the Dashboard Payload
Aggregate all processed customers into the structure below. For each customer:
- `name`: normalized company name (e.g. "ZenBusiness", not "airops-zenbusiness")
- `channel`: source Slack channel with `#` prefix
- `asana_ticket`: ticket name string
- `asana_ticket_id`: ticket GID (from `output/{date}/asana/tickets.json`)
- `issues`: full list of issues with all fields

### 2. Include Unmatched Customers
Customers with issues found in Slack but no matching Asana ticket go into `unmatched_customers[]`. This surfaces them for manual ticket creation.

### 3. Write the File
Write to `$DASHBOARD_OUTPUT_PATH` (set in `.env`).

**Schema:**
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
    {
      "name": "Fortinet",
      "channel": "#ext-airops-ping",
      "issues": []
    }
  ]
}
```

### 4. Confirm Write
Log `Dashboard data saved ✓` for inclusion in the Step 10 final report.
