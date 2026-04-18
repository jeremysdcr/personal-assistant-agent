# Boot-Sync Prompt Fragment

Embedded at the start of every CLI-side routine and referenced from CLAUDE.md Quick Start. This is the "fast path" reconciler that runs before any user-facing work when Jeremy is interacting via the CLI.

The authoritative reconciliation logic lives in `prompts/reconcile.md` (used by the Anthropic-infra routine). This fragment is the shorter CLI-side version — same intent, fewer ceremonies.

---

## Applies to

Surface A (CLI desktop extension) and Surface C (routines) — both have Notion MCP. **If you are on Surface B (cloud interactive, no MCP), short-circuit and skip the rest of this prompt.** Without MCP, boot-sync cannot drain `vault/cloud-actions.jsonl` or re-sync the cache, so running it is a no-op at best and misleading at worst.

Surface B short-circuit procedure:
1. Peek at `vault/cloud-actions.jsonl`. If non-empty, tell Jeremy "N cloud actions pending — next reconciler run or CLI boot will drain." If empty, no mention needed.
2. Proceed to user-facing work under Surface B's write rules (see CLAUDE.md → Surfaces and Write Rules): pivot to main before any state write, append intents to the cloud-actions log, never edit `vault/task-cache.json`.

Surface detection: try any Notion MCP tool (e.g. `notion-fetch` on the Daily Brief page). If it errors with "tool not available" or equivalent, you are on Surface B — stop here.

---

## Steps

### 0. Get real time
Run `TZ="America/New_York" date "+%Y-%m-%d %H:%M %Z"` to establish the actual Eastern time. Do not rely on the injected `currentDate`.

### 1. Pull latest (and pivot to main)

Routine workspaces may start the session on an auto-generated working branch (e.g. `claude/quirky-knuth-rGvoh`). Pivot to `main` explicitly so any later `git push` lands on `main` rather than a side branch:

- `git fetch origin main`
- `git checkout -B main origin/main` — forces local `main` to match `origin/main`, creating or resetting the branch as needed
- If you see conflicts later when writing state files: prefer remote (`git checkout --theirs`) for `task-cache.json`; investigate anything else

### 2. Drain `vault/cloud-actions.jsonl` (if non-empty)
Apply each entry to Notion via MCP. Follow the mapping in `prompts/reconcile.md` Step 3. Archive successful entries to `vault/cloud-actions-archive/YYYY-MM.jsonl` and rewrite the live file with only errored entries.

If Notion MCP is unavailable: do NOT drain. Leave the file intact. Tell Jeremy "X cloud actions pending, Notion MCP unavailable" and proceed with whatever cached data is available.

### 3. Re-sync Notion → cache (conditional)
Skip this step if ALL of:
- No actions were processed in step 2
- `synced_at` in `vault/task-cache.json` is within the last 15 minutes

Otherwise: query `notion-query-database-view` on `view://3441e696-29d1-81ac-84aa-000cd656c691`, overwrite `vault/task-cache.json` using the slim schema defined in `prompts/reconcile.md` step 5 (single source of truth).

### 4. Commit changes (if any)
If step 2 or 3 modified files:
- `git add vault/` and commit with message `boot-sync: applied {N} cloud actions` (or `boot-sync: cache resynced` if actions count = 0)
- Do not push — the commit will go out with whatever work comes next

### 5. Proceed
Now do the actual user-facing work (catch-up, briefing, etc.).

## When to skip the whole sequence

- If Jeremy asks a question that clearly doesn't depend on PA state (e.g. "explain this JSON") — just answer
- If this is a follow-up message in an active conversation and boot-sync already ran in a prior turn — don't re-run unless there's a reason (e.g. user says "re-sync" or enough time has passed that staleness matters)
