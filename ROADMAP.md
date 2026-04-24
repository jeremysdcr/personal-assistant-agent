# Roadmap — Structural PA Improvements

Living list of structural changes to the PA system. One-off tasks belong in PA Tracker; this file tracks work with design rationale attached.

Status values: `proposed` (needs Jeremy sign-off) → `planned` (approved, not started) → `in_progress` → `done`.

---

## Active

### 1. Fix the nightly carry-forward ratchet — `proposed`

**Location:** [prompts/eod-review.md](prompts/eod-review.md) Step 4 (lines 108–112)

**Problem:** Unconditional re-date of all `due_date = today AND status != done` items to tomorrow creates the "42 items dated today" pileup. No distinction between items Jeremy is driving, items waiting on someone else, and items that have been silently carried forward for weeks.

**Proposed change:** Conditional carry-forward based on status/type and carry-count (counted from "Carried forward from" stamps in Notes):

- `status=waiting` OR `type=commitment_theirs` → not carried day-to-day; date pushed to `today + 3 business days` with note "Nudge cadence — waiting on {person}"
- `carry_count >= 3` → status flipped to `stale`, surfaced in EOD brief's "Needs your decision" section, date NOT pushed
- Otherwise → normal 1-day carry forward

**Effect:** "Due today" bucket drops to 5–10 real items within a week. Waiting items appear every ~3 days instead of daily. Stuck items surface for triage instead of silently re-dating forever.

**Open question:** Is `carry_count >= 3` the right threshold? Could be 2 (more aggressive) or 5 (more forgiving).

**Priority:** highest leverage. Do first.

---

### 2. Add weekly triage routine — `proposed`

**Location:** new `prompts/weekly-triage.md`, registered as a routine

**Problem:** The first-principles task review (priority inflation, duplicate detection, dateless drift, recurring-pattern detection, proposed next-week schedule) is something that currently only happens when Jeremy asks for it manually. By then the noise is thick.

**Proposed change:** New routine, Sunday 8pm ET (1 run/week), that:

- Counts items by status/priority/due-date bucket
- Flags items with carry_count ≥ 3
- Flags items with `priority=high AND age>14d AND status=open` (priority inflation candidates)
- Flags items with `no due_date AND created>14d ago` (dateless drift)
- Detects recurring title patterns (same title on 7d cadence ≥ 3 times → recommend calendar migration)
- Detects potential duplicates (same Person + title-slug similarity ≥ 0.7, any age)
- Produces a proposed Mon–Fri schedule
- Writes output to a new Notion "Weekly Triage" DB page + phone push

Jeremy reviews in Notion; approved updates applied on next CLI boot.

**Open questions:**
- Cadence: Sunday 8pm ET, or Friday 3pm ET (catches drift before weekend), or both?
- Output: new Notion DB vs. append to daily journal?
- Should this include the priority-budget surface check (see #3) inline, or keep it in scan-and-check?

**Priority:** second-highest. Builds on #1 (the cleanup #1 enables makes #2's output actionable).

---

### 3. Priority-budget surface in scan-and-check — `proposed`

**Location:** [prompts/scan-and-check.md](prompts/scan-and-check.md) Part B, after Step 12

**Problem:** No feedback loop on priority inflation. The current state (49 high-priority items, 51% of active) is invisible until a manual review.

**Proposed change:** After per-item evaluation in Step 12, count `priority=high AND status in (open, in_progress)`. Emit a line into the Scan & Check journal section:

```
- Priority budget: {N}/15 high-priority items (over budget — drain needed)
```

Append to Step 14 push notification if `N > 18`. Same treatment for "due today" count > 12. No auto-demotion — just visibility.

**Open question:** Thresholds (15 high, 12 due-today) — feel right, or want to tune?

**Priority:** third. Cheap to ship, compounds with #1.

---

### 4. Extend dedup window beyond 48h — `proposed`

**Location:** [prompts/obligation-extract.md](prompts/obligation-extract.md) Section 6, rule 2 (lines 186–193)

**Problem:** Dedup rule 2 ("same person + similar title within 48 hours") misses duplicates created further apart. PA-72 (Apr 11) vs PA-5 (Apr 1) both track Bezede Project Elevate — 10 days apart, both live.

**Proposed change:** Extend the 48h window to 14 days. Add a soft-flag path: ambiguous matches (0.7–0.85 title similarity) get flagged for weekly-triage review rather than auto-skipped. False-dedup is worse than letting a possible dup through.

**Priority:** fourth. Cheap. Compounds with #2 (weekly triage catches the flagged ambiguous matches).

---

### 5. Split Must-Do / Waiting / Overdue in morning brief — `proposed`

**Location:** [prompts/morning-brief.md](prompts/morning-brief.md) Step 9, "Attention Required" section (line 131)

**Problem:** Overdue + due-today + urgent all render as one block of `- [ ]` checkboxes. An 18-item list reads as a fire when most of it is external-waiting or dateless drift.

**Proposed change:** Split the Attention Required section into three subsections:

```
## Must Do Today (items I'm driving)
{due_date = today AND type in (task, commitment_mine)}

## Waiting — Nudge If Silent (items I'm chasing)
{status = waiting OR type = commitment_theirs, with "Last nudged: {date}" inline}

## Overdue — Decide or Push
{due_date < today, with carry_count and suggested action}
```

**Priority:** fifth. Pure UX win. Do alongside #1 — the split is only meaningful once #1 has drained the false "due today" pile.

---

### 6. Recurring-item pattern detection — `proposed`

**Location:** part of weekly triage (#2)

**Problem:** "Date with Leore" (PA-148…PA-159, 12 weekly Monday rows through July) is a recurring calendar event that ended up as 12 tracker rows. No detection prevents the next one.

**Proposed change:** In weekly triage, detect clusters of items with identical titles on a 7-day cadence ≥ 3 times. Recommend migrating to a Google Calendar recurring event; propose archival of the rows on Jeremy's confirmation.

**Priority:** sixth. Lives inside #2 — not a standalone change.

---

## Done

_(none yet)_
