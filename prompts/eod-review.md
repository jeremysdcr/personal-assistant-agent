# EOD Review Routine Prompt

**Schedule:** `3 18 * * 1-5` (6:03pm ET weekdays)
**Model:** Sonnet

---

You are generating the end-of-day review for Jeremy Rosmarin (Sidecar Capital Partners). Today's date is {today}.

## Steps

### 0. Boot-sync
Run the boot-sync sequence from `prompts/boot-sync.md` before counting completions or carrying items forward — otherwise "completed today" counts can miss state changes made from cloud surfaces.

### 1. Load State
Clone repo. Read `vault/daily/{YYYY-MM-DD}.md` (today's accumulated journal). Read `vault/task-cache.json`.

### 2. Calendar Review
Use `gcal_list_events` for today. Note which meetings happened. Cross-reference with open items — were any items related to today's meetings completed? Also note any HARD OVERLAPs or TIGHT TRANSITIONs that actually occurred today (retrospective pattern signal — useful for catching recurring conflict types).

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
Also query Notion for open conflict items (Source = `calendar`, Source Ref starts with `conflict:`, Status in (open, in_progress, waiting, stale)) with Due Date = tomorrow. Do NOT run a fresh conflict sweep — the morning brief is the sole creator; this step only surfaces already-tracked conflicts so Jeremy sees them the night before instead of at 7am.
List tomorrow's meetings, conflicts, and due items.

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
{Tomorrow's calendar + items due tomorrow + any open conflict items due tomorrow}

### Stats
- Completed: {N}
- Created today: {N}
- Open: {N}
- Overdue: {N}
- Carried forward: {N}

### Extraction Quality
{False positive notes if any, otherwise "No same-day cancellations."}
```

### 8. Push Notification
Create a Notion comment on the Daily Brief page (ID: `3441e696-29d1-815b-9a43-c156cad7fc34`) via `notion-create-comment`. This is what Jeremy sees on his phone lock screen as the day wraps — keep it tight (3-5 lines):

- Lead with completion count + 1-2 highlights (e.g. "✅ 2 done today — PA-9 (Yossi booth), PA-30 (AHS tuition)")
- Carry-forward count if non-zero (e.g. "↪️ 1 carried to tomorrow: PA-36")
- Tomorrow's top 1-2 items + any calendar conflict (e.g. "Fri: PA-60 (INQ entity) + PA-57 (AAE pickup 5pm)")
- **If step 5 surfaced a new conflict for tomorrow, lead with it** (e.g. "⚠️ New Fri conflict: 10am Aucctus Board ↔ 11:20am Endo")

### 9. Update Notion and Cache
Update carried-forward items in Notion. Snapshot active items to `vault/task-cache.json`.

### 10. Commit and Push

You should already be on `main` from step 0's boot-sync. Run these three commands exactly:

```bash
git add vault/daily/ vault/task-cache.json
git commit -m "eod review: {YYYY-MM-DD}"
git push origin main
```

Substitute `{YYYY-MM-DD}` with today's date from step 0 (e.g. `2026-04-16`). Keep the commit message plain single-line — **do NOT** wrap it in a heredoc (`$(cat <<'EOF' ... EOF)`), add Claude Code attribution, or include any session link. The routine's author identity (`Claude <noreply@anthropic.com>`) is the audit trail.

## Setup Requirements
- Enable "Allow unrestricted branch pushes" for jeremysdcr/personal-assistant-agent
- All MCP connectors available by default (uses Calendar and Notion)
