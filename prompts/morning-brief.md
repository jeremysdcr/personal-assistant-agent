# Morning Brief Routine Prompt

**Schedule:** `57 6 * * *` (6:57am ET, every day including weekends)
**Model:** Sonnet

---

You are generating the morning briefing for Jeremy Rosmarin, who runs Sidecar Capital Partners (growth equity) and a consulting business. Today's date is {today}.

## Output discipline (hard rules — prior runs have hit `Stream idle timeout - partial response received`)

You run autonomously — Jeremy does not see your output. Emit tool calls, not explanations. Never narrate what you're about to do; do it. The reconciler's step 5 (`prompts/reconcile.md`) is the canonical version of this discipline; the steps below carry the same hazard and must be treated the same way:

- **Step 9 (daily journal Write):** After the last upstream read (gmail/gcal/HubSpot/Notion), the very next thing in your response must be the `Write` tool call for `vault/daily/{today}.md`. No preview of the brief, no enumerating items/meetings/attendees/deals in prose, no re-reading files first.
- **Step 12 (cache resync Write):** After `notion-query-database-view` returns, emit the `Write` on `vault/task-cache.json` immediately. No narration, no re-read, no intermediate file. Same transform as `prompts/reconcile.md` step 5.
- **Do NOT parallelize boot-sync's cache resync with Step 2's calendar fetch.** Boot-sync is a hard prerequisite; finish it (including its commit/push) before starting Step 1. Combining a failed cache resync with a 50KB+ calendar response in the same turn is the exact recovery scenario that blows the stream-idle budget.

If you catch yourself generating prose between an upstream tool result and the next tool call for any of these steps, stop and emit the tool call.

## Steps

### 0. Boot-sync
Before generating the brief, run the boot-sync sequence from `prompts/boot-sync.md`: pull, drain `vault/cloud-actions.jsonl` into Notion via MCP, resync cache if stale. This ensures cloud-captured state changes from Jeremy's phone/web sessions are reflected in today's brief.

### 1. Load State
Clone the repo (already done by step 0 if routine; otherwise clone now). Read `vault/task-cache.json` and `vault/key-relationships.md`.

### 2. Calendar Fetch (today + next 7 days)
Use `gcal_list_events` with timeMin: today 00:00 ET, timeMax: today + 7 days 23:59 ET.
For each event, note: time, title, attendees, location/link, response status, transparency.
Keep today's subset for the schedule section; keep the full 7-day list for Step 2.5.

### 2.5. Conflict Sweep (7-day window)
Detect conflicts across the full 7-day event list, then reconcile against existing Notion conflict items.

**Conflict rules:**
- **HARD OVERLAP** — two non-all-day events whose times intersect (`A.start < B.end AND B.start < A.end`). Exclude events Jeremy declined or with `transparency: transparent`.
- **TIGHT TRANSITION** — <5 min gap between consecutive events where at least one has a physical location AND the locations differ (physical↔virtual counts; two different physical addresses count).
- Ignore all-day events.

**For each detected conflict pair:**
1. Build stable `source_ref` = `conflict:{eventA_id}:{eventB_id}` (sort IDs lexically).
2. Query Notion for a PA Tracker item with this Source Ref (any status).
3. Apply dedup:
   - **No match** → `notion-create-pages` with:
     - Title: `Conflict: {MM/DD HH:MM} — {event A title} ↔ {event B title}`
     - Type: `follow_up`, Status: `open`, Source: `calendar`
     - Priority: `high` if HARD OVERLAP within 48h, `medium` if HARD OVERLAP beyond 48h, `low` if TIGHT TRANSITION
     - Due Date: date of the conflicting events
     - Source Ref: the `conflict:...` key above
     - Notes: conflict type, both event titles/times, attendees, locations
   - **Match with status in (open, in_progress, waiting, stale)** → skip; already tracked.
   - **Match with status in (done, cancelled)** → skip; Jeremy dismissed it — do not recreate.

**Resolution pass (after dedup):**
4. Query Notion for all PA Tracker items where Source = `calendar` AND Source Ref starts with `conflict:` AND Status in (open, in_progress, waiting, stale).
5. For each, re-check whether the pair still conflicts under the rules above (event IDs still exist, still overlap/tight, not declined). If the conflict no longer exists, `notion-update-page` to set Status = `done` and append to Notes: `Auto-resolved {YYYY-MM-DD}: conflict cleared`.

Track which items are newly created this sweep — the journal and push notification use that to flag `(new)`.

### 3. Calendar-to-Task Correlation
For each meeting attendee:
- Cross-reference against the Person field in `vault/task-cache.json` to find open items
- Surface related items inline with the calendar entry

### 4. Meeting Prep — HubSpot
For each meeting attendee:
- Search HubSpot contacts (`search_crm_objects`, objectType: "contacts", query: attendee name)
- Pull associated deals: stage, amount, recent activity
- Include deal pipeline context in the briefing

### 5. Meeting Prep — Google Drive
For each meeting with key relationship attendees:
- Use `search_files` to find recent documents mentioning the attendee or company name
- Use `read_file_content` on the most recently modified relevant doc
- Include a one-line summary

### 6. Overnight Email
Use `gmail_search_messages` with query `after:{yesterday 6pm ET, epoch seconds} (to:jeremy@sidecarcapitalpartners.com OR from:jeremy@sidecarcapitalpartners.com) -in:drafts`. **The `-in:drafts` is load-bearing: Gmail's `from:` clause otherwise matches unsent drafts, which get narrated as "Jeremy sent / replied" in the brief. See `scan-and-check.md` Step 5 for the full rationale.**
Classify the top 20 as ACTIONABLE / FYI / SKIP using the extraction framework in `prompts/obligation-extract.md`.
- Create Notion items for ACTIONABLE emails (use `notion-create-pages` with data source `b3e39150-8cf2-491f-b65f-f13f38fae886`)
- Highlight emails from key relationships separately

### 7. Active Deals Snapshot
Query HubSpot for active deals (`search_crm_objects`, objectType: "deals", filter: deal stage not in closed stages).
Summarize: deal count by stage, any deals with recent stage changes.

### 8. Active Items from Notion
Call `notion-query-database-view` with `view_url: https://www.notion.so/3441e69629d1815b9a43c156cad7fc34?v=3441e69629d181ac84aa000cd656c691` (the Active Items view — use this exact URL string; `notion://...` and `view://...` shorthands are rejected with `validation_error`).
Group into:
- Overdue (due_date < today)
- Due today
- Due this week
- My commitments to others (type = commitment_mine)
- Waiting on others (type = commitment_theirs or follow_up)

### 9. Write Daily Journal
Write `vault/daily/{YYYY-MM-DD}.md`:

```markdown
# Daily Brief — {Day of Week, Month DD, YYYY}

## Today's Schedule
{Calendar items with times, attendees, deal context, Drive docs, and correlated open items}

## Calendar Conflicts (next 7 days)
{All open conflict items (source_ref starts with `conflict:`), sorted by due date. Flag newly created ones with `(new)`. Omit this section entirely if no open conflict items exist.}

## Deal Pipeline Snapshot
{Active deals by stage, recent stage changes}

## Attention Required
{Overdue items, items due today, urgent items}

## Key Relationship Emails
{Emails from key contacts — highlighted separately}

## Overnight Email Summary
{Remaining actionable and FYI items}

## Active Commitments
### I Owe
{commitment_mine items sorted by due date}

### Waiting On
{commitment_theirs and follow_up items sorted by due date}

## Open Tasks
{Remaining open tasks by priority}

## FYI
{Notable informational items}
```

### 10. Push Notification
Create a Notion comment on the Daily Brief page (ID: `3441e696-29d1-815b-9a43-c156cad7fc34`) via `notion-create-comment`:
- 3-5 bullet points: schedule highlights, deal updates, attention items, key emails
- **If any HARD OVERLAP was newly created this sweep within the next 48h, lead with it.** (e.g. "⚠️ New conflict: Tue 2pm Acme ↔ Tue 2:30pm Smith")
- Keep it concise — this is what Jeremy sees on his phone lock screen

### 11. Gmail Draft
Create a draft email to jeremy@sidecarcapitalpartners.com via `gmail_create_draft`:
- Subject: "Daily Brief — {Month DD, YYYY}"
- Body: the full briefing text
- This is a reference copy in Gmail (does not push-notify)

### 12. Update Cache
Call `notion-query-database-view` with `view_url: https://www.notion.so/3441e69629d1815b9a43c156cad7fc34?v=3441e69629d181ac84aa000cd656c691` (same URL as Step 8), write to `vault/task-cache.json` using the slim schema in `prompts/reconcile.md` step 5.

### 13. Commit and Push

You should already be on `main` from step 0's boot-sync. Run these three commands exactly:

```bash
git add vault/daily/ vault/task-cache.json
git commit -m "daily brief: {YYYY-MM-DD}"
git push origin main
```

Substitute `{YYYY-MM-DD}` with today's date from step 0 (e.g. `2026-04-16`). Keep the commit message plain single-line — **do NOT** wrap it in a heredoc (`$(cat <<'EOF' ... EOF)`), add Claude Code attribution, or include any session link. The routine's author identity (`Claude <noreply@anthropic.com>`) is the audit trail.

## Setup Requirements
- Enable "Allow unrestricted branch pushes" for jeremysdcr/personal-assistant-agent
- All MCP connectors available by default (uses Gmail, Calendar, Drive, Notion, HubSpot)
