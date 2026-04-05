# Step 5: Fetch Asana Tickets Assigned to User

## Goal
Pull all open Asana tickets assigned to `$ASANA_USER_ID` in project `$ASANA_PROJECT_ID` to use as the target for Slack issue mapping and comment writing.

All credential values are read from `.env` / `.env`.

## Inputs
- `$ASANA_USER_ID` — from `.env`
- `$ASANA_PROJECT_ID` — from `.env`
- `$ASANA_PAT` — from `.env`

## Actions

### 1. Authenticate with Asana
- Use `$ASANA_PAT` as a Bearer token
- Base URL: `$ASANA_BASE_URL` (default: `https://app.asana.com/api/1.0`)

### 2. Fetch Tasks in Project
Call:
```
GET /projects/$ASANA_PROJECT_ID/tasks
  ?assignee=$ASANA_USER_ID
  &opt_fields=gid,name,assignee,completed,notes,memberships,created_at,modified_at
  &completed_since=now
```
- `completed_since=now` returns only incomplete tasks
- Paginate using `offset` if `next_page` is returned

### 3. Extract Company Name from Ticket Title
- Each ticket title is expected to contain a `{companyname}` identifier
- Apply the same normalization used for Slack channel names:
  - Lowercase
  - Strip punctuation / spaces → slugified form
  - Example: `"Acme Corp - CMS Integration"` → `acmecorp` or `acme-corp`
- Store as `ticket.company_slug` for matching in Step 6

### 4. Fetch Existing Comments (to avoid duplicates)
For each ticket, call:
```
GET /tasks/{task_gid}/stories
  ?opt_fields=gid,type,text,created_at,created_by
```
- Filter for `type: "comment"` stories
- Store the most recent comment's `created_at` as `last_comment_at`
- Store full comment texts to check for duplicate content in Step 7

## Output
Write to `output/{date}/asana/tickets.json`:
```json
[
  {
    "gid": "9876543210",
    "name": "Acme Corp - CMS Integration",
    "company_slug": "acmecorp",
    "completed": false,
    "last_comment_at": "2026-04-01T12:00:00Z",
    "existing_comment_texts": ["Previous sync comment..."]
  }
]
```
