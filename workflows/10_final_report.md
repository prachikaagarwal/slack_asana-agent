# Step 10: Final Sync Report

## Goal
After all steps complete, output a concise summary of everything the sync did.

## Inputs
- `output/{date}/asana/posted_comments.json`
- `output/{date}/asana/stale_nudges.json`
- `output/{date}/asana/unmatched_channels.json`
- Aggregated issue counts from Steps 4–7

## Output Format

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

## Rules
- If no issues were found for a customer, omit them from the updated list
- If no stale nudges were posted, omit that section
- If dashboard write failed, report `Dashboard data save FAILED: [error]` instead of ✓
- Keep the report scannable — readable in under 30 seconds
