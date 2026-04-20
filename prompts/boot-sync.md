# Boot-Sync Prompt Fragment

Embedded at the start of every CLI-side routine and referenced from CLAUDE.md Quick Start. This is the "fast path" reconciler that runs before any user-facing work when Jeremy is interacting via the CLI or cloud.

The authoritative reconciliation logic lives in `prompts/reconcile.md` (used by the Anthropic-infra routine). This fragment is the shorter interactive version — same intent, fewer ceremonies.

Boot-sync has two independent phases:

- **Phase A — Workspace refresh.** Cheap, network-only. Pulls latest `main` into the local workspace so every subsequent read (`vault/task-cache.json`, `vault/daily/*`) reflects whatever the other surface just wrote. Runs on every turn, every surface — never skipped except for the narrow meta-question escape hatch below.
- **Phase B — MCP reconcile.** Drains `vault/cloud-actions.jsonl` into Notion and resyncs the cache. Requires Notion MCP. Surface B (cloud interactive) skips Phase B entirely and does a queue peek instead.

The single bug this structure is designed to prevent: cloud sessions reading a workspace snapshot that's hours behind `origin/main` because the earlier Surface-B short-circuit also threw away the refresh step.

---

## Phase A — Workspace refresh (every turn, every surface)

### 0. Get real time
Run `TZ="America/New_York" date "+%Y-%m-%d %H:%M %Z"` to establish the actual Eastern time. Do not rely on the injected `currentDate`.

### 1. Safety check — do not silently destroy work

Before any fetch/reset, check the local workspace:

- `git status --porcelain` — if non-empty, **stop**. Tell Jeremy "uncommitted changes in workspace — commit or stash before I sync" and wait. Surface B rules require commit-immediately on every write, so this should not trip; if it does, a silent reset would erase work.
- Remember the current HEAD sha (for the "pulled N commits" line later): `PRE_HEAD=$(git rev-parse HEAD)`.

### 2. Fetch

- `git fetch origin main`.
- If the fetch errors (network unreachable, auth, etc.), **do not reset**. Print `"couldn't reach origin; working from local snapshot last synced at <last-commit-ts>"` and proceed to Phase B. The session will run on whatever is already checked out.

### 3. Unpushed-commit check

- `git rev-list --count origin/main..HEAD` — commits on the current branch not yet on `origin/main`.
- If non-zero, **stop**. Tell Jeremy "local commits ahead of origin — push first, then I'll re-sync" and wait. (Routine workspaces often start on an auto-generated `claude/<slug>` branch; a non-zero count here usually means a prior turn committed without pushing.)

### 4. Pivot to main

- `git checkout -B main origin/main` — forces local `main` to match `origin/main`, switching off any working branch the harness spawned us on. After this, all later writes land on `main`.
- If HEAD moved (`$(git rev-parse HEAD) != $PRE_HEAD`), print `"pulled N commits from main since last turn"` where N = `git rev-list --count $PRE_HEAD..HEAD`. Suppress the line when N = 0 (silent sync is fine).

---

## Phase B — MCP reconcile (Surface A/C only; skippable when clean)

### 5. Surface detect

Try any Notion MCP tool (e.g. `notion-fetch` on the Daily Brief page `3441e696-29d1-815b-9a43-c156cad7fc34`). If it errors with "tool not available" or equivalent, you are on **Surface B** — skip to step 9 (queue peek).

### 6. Drain `vault/cloud-actions.jsonl` (if non-empty)

Apply each entry to Notion via MCP. Follow the mapping in `prompts/reconcile.md` Step 3. Archive successful entries to `vault/cloud-actions-archive/YYYY-MM.jsonl` and rewrite the live file with only errored entries.

If a Notion MCP call fails mid-drain (transient error, not "tool not available"): leave the entry in place with an `error_flag` and continue with the next entry. Do not block the whole catch-up on one bad entry.

### 7. Re-sync Notion → cache (conditional)

Skip this step if ALL of:
- No actions were processed in step 6
- `synced_at` in `vault/task-cache.json` is within the last 15 minutes
- Phase A did not pull any new commits (if it did, main may have newer cache state worth reading but not re-querying)

Otherwise: call `notion-query-database-view` with `view_url: https://www.notion.so/3441e69629d1815b9a43c156cad7fc34?v=3441e69629d181ac84aa000cd656c691` (the Active Items view — use this exact URL string; `notion://...` and `view://...` shorthands are rejected with `validation_error`), then snapshot to `vault/task-cache.json` via the shell-offload pattern below. The slim schema is defined authoritatively in `prompts/reconcile.md` step 5.

**Retry on timeout.** The Active Items view returns a heavy payload (~50KB+, all Notes blobs included regardless of view column visibility — see PA-98) and the MCP transport intermittently times out. If the `notion-query-database-view` call returns `API Error`, `Request timed out`, or any equivalent transport failure, retry it **once** after a 30-second pause (`sleep 30` via Bash). Do not retry more than once.

**If both attempts fail:** do not write the cache. Emit a `notion-create-comment` on the Daily Brief page `3441e696-29d1-815b-9a43-c156cad7fc34` with body `"Boot-sync: cache resync failed — Notion query timed out twice. Cache last refreshed {synced_at from existing vault/task-cache.json}. Routine continuing with stale cache."`. Then continue to step 8. A stale cache is degraded but workable for one routine pass; the next run will retry.

**On success — shell-offload pattern.** Mirror `prompts/scan-and-check.md` step 15 exactly. Emit two tool calls back-to-back, no prose between:

**7a.** `Write` to `vault/.cache-raw.json` with the `notion-query-database-view` tool result verbatim — the object containing `results` and `has_more`. This is a pure byte copy: no transform, no filtering, no reasoning about items.

**7b.** `Bash` with exactly this one-liner (`jq` 1.6+ is available on routine infra):

```
jq --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" '{synced_at: $ts, items: [.results[] | {item_id: ("PA-" + (."Item ID"|tostring)), title: .Title, type: .Type, status: .Status, priority: .Priority, due_date: (."date:Due Date:start" // null), person: (.Person // ""), notion_page_id: (.url | split("/") | last | split("?")[0] | gsub("-";"")), source_ref: (."Source Ref" // "")}]}' vault/.cache-raw.json > vault/task-cache.json && rm vault/.cache-raw.json
```

The 9-field projection is the canonical one from [prompts/reconcile.md](reconcile.md) step 5, expressed as a shell transform so no model reasoning is required per item. Tradeoff: same as scan-and-check — cache reflects state as of the query response, not later in-turn writes; next run reconciles.

**Output discipline (hard rules — prior runs have violated these and hit `Stream idle timeout - partial response received`):**

1. After the `notion-query-database-view` tool-result arrives, the **very next thing** in your response must be the `Write` call for `vault/.cache-raw.json` (step 7a), immediately followed by the `Bash` jq call (step 7b). No text between, no narration, no count, no enumeration of items.
2. Do **not** re-read `vault/task-cache.json` after step 7b. The shell command's success exit is sufficient. Self-verification burns the remaining turn budget and has caused idle timeouts.
3. If you find yourself generating prose between the MCP tool-result and step 7a, or between 7a and 7b, you are violating this rule — stop and emit the next tool call immediately.

### 8. Commit changes (if any)

If step 6 or 7 modified files:
- `git add vault/` and commit with message `boot-sync: applied {N} cloud actions` (or `boot-sync: cache resynced` if actions count = 0).
- **Push immediately:** `git push origin main`. The boot-sync commit is self-contained state and should ship on its own turn; deferring creates strand risk if the session ends without another state-change turn (the session Stop hook in `.claude/settings.json` will flag any unpushed commits as a safety net, but pushing here is the clean invariant).

### 9. Queue peek (Surface B only)

If you reached here via the Surface B short-circuit in step 5: open `vault/cloud-actions.jsonl` and count lines. If non-empty, tell Jeremy `"N cloud actions pending — next reconciler run or CLI boot will drain."` If empty, no mention needed.

### 10. Proceed

Now do the actual user-facing work (catch-up, briefing, etc.).

---

## When to skip phases

- **Skip everything (both phases) only when:** Jeremy asks a question that clearly doesn't depend on PA state (e.g. "explain this JSON", "how does the cache work"). Just answer.
- **Skip Phase B only (still run Phase A):** follow-up message in an active conversation where ALL of: Phase A just pulled zero new commits in this turn OR the prior turn, cache `synced_at` is within 15 minutes, and `vault/cloud-actions.jsonl` is empty. Phase A is always cheap enough to run — it's the read-side guarantee.
- **Never skip Phase A on a catch-up / briefing request.** Catch-ups are exactly when cross-surface freshness matters most.
