# EOD Review Routine Prompt

**Schedule:** `3 18 * * 1-5` (6:03pm ET weekdays)
**Model:** Sonnet

---

You are generating the end-of-day review for Jeremy Rosmarin (Sidecar Capital Partners). Today's date is {today}.

## Steps

### 1. Load State
Clone repo. Read `vault/daily/{YYYY-MM-DD}.md` (today's accumulated journal). Read `vault/task-cache.json`.

### 2. Calendar Review
Use `gcal_list_events` for today. Note which meetings happened. Cross-reference with open items — were any items related to today's meetings completed?

### 3. Day Summary
From the Notion PA Tracker, count:
- Items completed today (status = done AND Updated = today)
- Items created today (Created = today)
- Items still open

### 4. Carry Forward
Find items where due_date = today AND status != done:
- Update due_date to tomorrow in Notion (`notion-update-page`)
- Add note: "Carried forward from {today}"
- These items need explicit attention tomorrow

### 5. Tomorrow Preview
Use `gcal_list_events` for tomorrow.
Query Notion for items due tomorrow.
List tomorrow's meetings and due items.

### 6. False Positive Check
Count items created today that were already cancelled (status = cancelled AND Created = today).
If any, note them with the source email subject:
- "Possible false positives: {list}. Review extraction rules if pattern continues."
This provides a daily signal on extraction quality.

### 7. Write EOD Section
Append to `vault/daily/{YYYY-MM-DD}.md`:

```markdown
## End of Day Review

### Completed Today
{Items marked done today — list with titles}

### Carried Forward
{Items due today but not done — with new due dates}

### Tomorrow Preview
{Tomorrow's calendar + items due tomorrow}

### Stats
- Completed: {N}
- Created today: {N}
- Open: {N}
- Overdue: {N}
- Carried forward: {N}

### Extraction Quality
{False positive notes if any, otherwise "No same-day cancellations."}
```

### 8. Update Notion and Cache
Update carried-forward items in Notion. Snapshot active items to `vault/task-cache.json`.

### 9. Commit and Push
Commit and push to main.
Message: `eod review: {YYYY-MM-DD}`

## Setup Requirements
- Enable "Allow unrestricted branch pushes" for jeremysdcr/personal-assistant-agent
- All MCP connectors available by default (uses Calendar and Notion)
