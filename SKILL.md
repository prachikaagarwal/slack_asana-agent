---
name: customer-slack-to-asana
description: >
  Daily sync of customer Slack feedback to Asana tickets. Scans #ext-[company], #airops-[company], and #ext-airops-[company] channels for the last 3 days, extracts integration issues per client (pain point, current status, ideal status), detects resolution from the user's replies or emoji reactions, and writes ONE consolidated comment per Asana ticket. Also saves dashboard_data.json and posts stale ticket nudges. Trigger when the user asks to "scan customer channels", "sync slack to asana", "check what customers need", "what did customers say this week", or when the scheduled 9am daily trigger fires.
---

# Customer Slack → Asana Daily Sync

Full implementation is defined across two files — read both before running:

- **`CLAUDE.md`** — project rules, credential variable reference, and all 10 steps
- **`workflows/`** — one detailed workflow file per step (01–10)

Credentials and runtime config are loaded from **`.env`** (copy from `.env.example`).
