# EOD Review Routine Prompt

**Schedule:** `3 18 * * 1-5` (6:03pm ET weekdays)
**Model:** Sonnet

---

You are generating the end-of-day review for Jeremy Rosmarin (Sidecar Capital Partners). Today's date is {today}.

## Output discipline (hard rules — prior runs have hit `Stream idle timeout - partial response received`)

You run autonomously — Jeremy does not see your output. Emit tool calls, not explanations. Never narrate what you're about to do; do it. The reconciler's step 5 (`prompts/reconcile.md`) is the canonical version of this discipline; four steps below carry the same hazard and must be treated the same way:

- **Step 3.5 (meeting-notes create loop):** After the per-meeting parse and payload enumeration is done, the very next thing in your response must be the first `notion-create-pages` call on the PA Tracker data source. No prose listing what you're about to create. One call per item, back to back. Do not narrate between them.
- **Step 4 (carry-forward loop):** After querying Notion for items with `due_date = today AND status != done`, the very next thing in your response must be the first `notion-update-page` call. No prose enumerating what you're about to update. One tool call per carry-forward, back to back. Do not narrate between them.
- **Step 7 (EOD section Write):** After the last upstream read (gcal/Notion counts), the very next thing in your response must be the `Write` tool call appending to `vault/daily/{today}.md`. No preview of the text, no draft, no re-reading the file first.
- **Step 9 (cache resync Write):** After `notion-query-database-view` returns, emit the `Write` on `vault/task-cache.json` immediately. No narration, no re-read, no intermediate file. Same transform as `prompts/reconcile.md` step 5.

If you catch yourself generating prose between an upstream tool result and the next tool call for any of these steps, stop and emit the tool call.

## Steps

### 0. Boot-sync
Run the boot-sync sequence from `prompts/boot-sync.md` before counting completions or carrying items forward — otherwise "completed today" counts can miss state changes made from cloud surfaces.

### 1. Load State
Clone repo. Read `vault/daily/{YYYY-MM-DD}.md` (today's accumulated journal). Read `vault/task-cache.json`.

### 2. Calendar Review
Fetch today's events from the 4-calendar set defined in CLAUDE.md → MCP Tools → Google Calendar → "Calendar set". Emit 4 `gcal_list_events` calls in parallel (one per calendar role: `primary`, `personal`, `family`, `holidays`); tag each event with its `_calendar_role` and merge.

- **Primary events** — note which meetings happened. Cross-reference with open items — were any items related to today's meetings completed? Also note any HARD OVERLAPs or TIGHT TRANSITIONs that actually occurred today between {primary ∪ personal} (retrospective pattern signal — useful for catching recurring conflict types).
- **Personal events** — note which happened (folds into Step 7's recap if Jeremy wants to look back).
- **Family events** — informational only; if any overlapped a primary/personal meeting, note as a soft flag in Step 7 ("⚠️ today's 3pm Acme overlapped [Family] Eitan soccer — did you attend the family event?").
- **Holiday events** — if today was a Jewish holiday, note it in Step 7's banner.

### 3. Day Summary
From the Notion PA Tracker, count:
- Items completed today (status = done AND Updated = today)
- Items created today (Created = today)
- Items still open

### 3.5. Mirror Meeting-Notes Action Items

Pull unchecked action items from recent Meeting Notes DB rows into PA Tracker so briefs, carry-forward, and crack-check include them. One-way create-only mirror — no writeback to Meeting Notes DB, no updates after create.

**Read watermark.** Fetch the Meeting Notes Scan Marker config page directly: `notion-fetch` on page ID `34b1e69629d181e98338f30a989170fa`. Parse ISO timestamp from the `Notes` property. If the fetch fails (page archived/missing), fall back to querying PA Tracker data source `collection://b3e39150-8cf2-491f-b65f-f13f38fae886` for Type = `config`, Title = `Meeting Notes Scan Marker`. If still absent, default watermark to 24h ago and re-create the row at the end of this step.

**Query new/edited meeting notes.** Call `notion-query-meeting-notes` with a single property filter:

```
{"property": "last_edited_time", "filter": {"operator": "date_is_after", "value": {"type": "exact", "value": "{watermark ISO string}"}}}
```

The tool defaults to Jeremy-as-attendee/creator — no extra scoping needed.

**Per meeting note, fetch and parse the body.** Call `notion-fetch` on the page ID. In the returned `<content>` block, normalize `<br>` → `\n` (Fathom-formatted notes use `<br>` as the line separator). Then apply two parsing strategies and collect items from both into the same pool — dedup is by slug-based source_ref, so any overlap is a no-op.

**Strategy A — Fathom "Next Steps" format (primary for recent notes).** Look for a line matching `^Next Steps\s*$` (case-insensitive). If found, treat all lines after it as the action region (Fathom puts Next Steps at the tail of the body). Within that region, recognize two sub-styles that may coexist in the same note:

- **Hierarchical (Style A):** a line matching `^\s{2}- (?P<who>[^:]+):\s*$` (no text after the colon) opens a person block. Following lines matching `^\s{4,}- (?P<text>.+)$` (deeper indent) are actions for that person. Block ends at the next Style A header or a dedent to `^\s{2}- `.
- **Flat with prefix (Style B):** a line matching `^\s{2}- (?P<who>[^:]+):\s+(?P<text>.+)$` (text after the colon) is a single action for that person.

**Strategy B — Markdown / `###`-heading format (fallback for hand-structured notes).** Scan for `###`- or `##`-level headings matching the action-signaling regex set below (case-insensitive):

| Heading pattern | PA `Type` | Person override |
|---|---|---|
| `^next actions?(\s*\((?P<who>[^)]+)\))?$` | `task` if `who` ≈ Jeremy/empty, else `commitment_theirs` | `who` if not Jeremy |
| `^action items?$` | `task` | — |
| `^follow[- ]?ups?(\s+expected)?(\s*\((?P<who>[^)]+)\))?$` | `follow_up` | `who` if present |
| `^waiting on\s+(?P<who>.+)$` | `follow_up` | `who` |
| `^items? to send to\s+(?P<who>.+)$` | `task` | `who` |
| `^to[- ]?dos?$` | `task` | — |

Under a matching heading, collect only `- [ ] {text}` lines (unchecked). Skip `- [x]` (in-meeting completions) and plain `- {text}` bullets. Stop at the next `###`/`##` heading.

**Jeremy-identity synonyms (case-insensitive).** Any of these in the `who` slot means the action is Jeremy's: `Jeremy`, `Jeremy Rosmarin`, `Sidecar`, `Sidecar Capital`, `Sidecar Capital Partners`, `Cyber Capital`, `Cyber Capital Partners`, `Both`. All other `who` values are counterparty-owed.

**Type assignment.** Jeremy-owned with a named counterparty → `commitment_mine` (this is the "what do I owe [person]" bucket queryable per CLAUDE.md). Jeremy-owned without a counterparty (rare for meeting notes) → `task`. Counterparty-owed → `commitment_theirs`. Strategy B follow-up/waiting headings keep their native `follow_up` mapping.

**Person field (counterparty on the PA item).**
- For Jeremy-owned items: extract from the meeting title. Split off the ` - {date}` suffix, then take the first name-like token that isn't Jeremy. For titles like `X and Y and Jeremy Rosmarin`, this yields `X`. For ambiguous group titles (`John, Rob and the CEO`), take the first token (`John`). Skip generic descriptors (`the founder`, `the CEO`, `the capital partner`, `the group`).
- For counterparty-owed items: the `who` name from the block.
- Fallback to blank if nothing else works.

**Build payloads.** For each collected checkbox / action line:

- `slug = lowercase(first_40_chars(text)).replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')`
- `source_ref = "meeting:" + meeting_page_id_no_dashes + ":" + slug`
- Skip if action text is empty/whitespace-only.
- **Dedup check (two-pass).** For each cached PA item (from `vault/task-cache.json`), SKIP the candidate if any of the following match:
  1. Exact match: cached `source_ref == candidate source_ref` (new format).
  2. **Legacy fallback (belt-and-suspenders):** cached `source_ref` contains `meeting_page_id_no_dashes` (handles pre-convention items that stored just the bare page ID, e.g. `Source Ref = "3481e69629d181dab612db619e45b454"` on PA-120/121) **AND** the candidate's slug is a prefix-or-suffix substring of the cached item's title's slugified first 40 chars. This catches duplicate meeting items created by prior ad-hoc extraction paths that pre-date the `meeting:{id}:{slug}` convention.

PA Tracker payload:
- Title: action text, trimmed, trailing period stripped, truncated to 80 chars
- Type: per heading table (Jeremy-owned with counterparty → `commitment_mine`)
- Status: `open`
- Priority: `high` if text contains `urgent`, `today`, or `ASAP` (case-insensitive), else `medium`
- Due Date: today's date if text contains `today`; parsed ISO date if text has `by {weekday}` / `next {weekday}` / explicit `Month Dth`; else null
- Person: per heading table
- Source: `meeting_notes`
- Source Ref: as computed above
- Source Subject: parent meeting's `Meeting` property
- Notes: `From meeting: [{Meeting title}]({meeting_url}) ({Date})` using the clickable permalink `https://www.notion.so/{meeting_page_id_no_dashes}`. **If the meeting's `Summary` page property is non-empty, append two newlines and then the Summary text verbatim** — this gives downstream briefings enough context to disambiguate a terse action title.

**Write to Notion.** After the full enumeration across all fetched meeting notes is complete, emit `notion-create-pages` on PA Tracker data source `collection://b3e39150-8cf2-491f-b65f-f13f38fae886` back-to-back, one page per item. No prose between calls — see the output discipline section above (this step is on the same hazard list as Step 4).

**Advance watermark.** After the last create (or immediately if zero items were generated), `notion-update-page` on page ID `34b1e69629d181e98338f30a989170fa` with command = `update_properties`, properties = `{"Notes": "<current ISO timestamp>"}`. If that page was missing at the start of the step, `notion-create-pages` on the PA Tracker data source instead, with Title = `Meeting Notes Scan Marker`, Type = `config`, Status = `open`, Priority = `low`, Source = `manual`, Notes = current ISO timestamp.

**Stage journal output for Step 7.** Remember the count of mirrored items and their titles — they go into a new `### Mirrored From Meetings` subsection (only if non-zero) and a `Meeting-notes tasks mirrored: {N}` line under `### Stats` (omit if zero).

### 4. Carry Forward
Find items where due_date = today AND status != done:
- Update due_date to tomorrow in Notion (`notion-update-page`)
- Add note: "Carried forward from {today}"
- These items need explicit attention tomorrow

### 5. Tomorrow Preview
Fetch tomorrow's events from the 4-calendar set (CLAUDE.md "Calendar set"). 4 parallel `gcal_list_events` calls, tag with `_calendar_role`, merge.
Query Notion for items due tomorrow.
Also query Notion for open conflict items (Source = `calendar`, Source Ref starts with `conflict:`, Status in (open, in_progress, waiting, stale)) with Due Date = tomorrow. Do NOT run a fresh conflict sweep — the morning brief is the sole creator; this step only surfaces already-tracked conflicts so Jeremy sees them the night before instead of at 7am.
Compute the same family-overlap awareness annotations as morning-brief Step 2.6 against tomorrow's primary/personal events for journal display.
List tomorrow's meetings (split by role), conflicts, due items, and any holiday landing on tomorrow.

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

### Mirrored From Meetings
{Step 3.5 items — omit section entirely if zero}

### Tomorrow Preview
{Holiday banner — `📅 Tomorrow: {holiday title}` if any `holidays`-role event covers tomorrow; omit otherwise.}
{Primary events for tomorrow with times. For any event carrying a family-overlap annotation, render an indented `⚠️ overlaps [Family] {event} {HH:MM–HH:MM}` line beneath it (same convention as morning-brief Step 9).}
{Personal & Family subsection: tomorrow's events with `_calendar_role` ∈ {`personal`, `family`}, prefixed `[Personal]` / `[Family]`. Omit subsection if empty.}
{Items due tomorrow and any open conflict items due tomorrow.}

### Stats
- Completed: {N}
- Created today: {N}
- Open: {N}
- Overdue: {N}
- Carried forward: {N}
- Meeting-notes tasks mirrored: {N}  (omit line if zero)

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
