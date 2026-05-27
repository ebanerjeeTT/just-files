# Bug Escapes — Project Plan

**Repo:** `tenstorrent/tt-auto-triage`, branch `ebanerjee/bug-escapes`
**Also read:** `/workspace/group/soft-objectives.md` — cost and backend preferences that should guide all development decisions
**Trigger workflow:** `tenstorrent/tt-metal`, branch `ebanerjee/bug-escapes`, `.github/workflows/triage-ci.yaml`

---

## Objective

Automatically detect tests that were failing on a feature branch, slipped through the merge gate, and are now failing on `main`. These are *bug escapes* — regressions that landed undetected.

The system should:
1. Find sustained failures on main (not flakiness, not infra noise)
2. Find the commit that *fixed* each failure (to identify which PR caused it and who owns it)
3. Classify the escape by type
4. Verify the fix is real by re-running CI — but *only the failing job*, not the entire workflow
5. Surface the results for human action

**Out of scope:** Issue creation is a separate project. This workflow does not create GitHub issues.

---

## Pipeline Phases

### Phase 0 — Smoke Test
- Check `envsubst` available
- Check LLM CLI available (branches on `LLM_BACKEND`: checks `copilot --version` or `agent --version`)
- Check `gh` authentication

### Phase 1 — Discovery
- Enumerate all GitHub Actions workflows in `tenstorrent/tt-metal`
- Classify each as `merge-gate`, `pr-gate`, or `other`
- Only `other` workflows are scanned for escapes (merge-gate and pr-gate are excluded)

### Phase 2 — Candidate Failures
- For each `other` workflow, fetch recent runs on `main` branch
- Find jobs with `CONSECUTIVE_RUNS` (default 3) or more consecutive failures
- Pre-filter: skip jobs with high run-to-run variance (likely flaky)
- Filter: exclude infrastructure errors (`InfraErrorV1.*` failure signatures)
- For remaining candidates: download failure logs (tail, not head — errors appear at end)
- Use LLM to confirm determinism: is this a real repeatable failure or infrastructure noise?
- **Batch size must be kept small** (~20 candidates per LLM call) to avoid timeouts — especially critical for copilot backend which is slower than cursor on large prompts

### Phase 3 — Fix Points
- For each confirmed candidate: find the commit on `main` that resolved the failure
- This is the *fix commit* — it tells us which PR introduced the regression
- Output: fix commit SHA, PR reference, affected files

### Phase 4 — Classification
- Classify each escape as:
  - `vertical` — test layer and code layer are owned by the same team
  - `horizontal` — cross-team impact
  - `cross_layer` — spans multiple hardware/software layers
  - `unknown` — insufficient signal
- Assign confidence: `high`, `medium`, `low`
- Deduplicate: if multiple jobs share the same root failure, merge into one escape entry

---

## Verification Step

- Triggered automatically after detection if `auto-verify=true`
- Only `vertical` + `high` confidence escapes with a valid fix commit SHA are verified
- **Critical:** only the specific failing job is re-run, not the entire workflow — prune the matrix/job list to that one job
- CI runs are dispatched on the fix commit to confirm the fix is real
- Budget cap: `verify-budget-hours` limits how long verification runs are dispatched
- Mock mode: `mock-verify=true` writes fake results without dispatching real CI (use during dev/testing)

---

## LLM Backend Support

Controlled by `LLM_BACKEND` env var (default: `cursor`).

| Backend | CLI | API endpoint |
|---|---|---|
| `cursor` | `agent --trust --model auto -p "$prompt"` | `api.cursor.sh` |
| `copilot` | `copilot -p "$prompt" --allow-all-tools` | `models.inference.ai.azure.com` |

- All LLM calls go through `bug-escapes/lib/cursor_agent.sh` — function names kept as `cursor_agent_*` for backward compat, but internally branch on `LLM_BACKEND`
- `check_failure_is_real()` in `verify/lib/verify_common.sh` also branches on `LLM_BACKEND`
- Secrets: `CURSOR_API_KEY` for cursor, `COPILOT_GITHUB_TOKEN` / `COPILOT_PAT` for copilot

---

## Inputs (triage-ci.yaml → bug-escapes-ci.yaml)

| Input | Default | Description |
|---|---|---|
| `llm-backend` | `cursor` | `cursor` or `copilot` |
| `lookback-days` | `14` | How far back to scan for failures — hourly cron uses `2` |
| `incremental-days` | `0` | Only surface escapes whose fix landed in last N days (0 = all) |
| `consecutive-runs` | `3` | Min consecutive failures to qualify as a candidate |
| `auto-verify` | `false` | Automatically dispatch verification after detection |
| `verify-budget-hours` | `6` | Max hours to spend dispatching verification runs |
| `mock-verify` | `false` | Dry-run verification without real CI |
| `max-phase` | `4` | Stop after this phase (useful for debugging) |

---

## Session Rules

- Each session *must end with at least one committed code change* to `ebanerjee/bug-escapes` in either tt-auto-triage or tt-metal. No session ends without shipping something.
- Sessions run hourly (cron) or immediately when triggered by `RESUME BUG ESCAPE WORK` in chat.
- A manual RESUME session that overlaps with the scheduled hour counts as that hour's work — don't double-run.
- `PAUSE BUG ESCAPES WORK` — finish the current task, then disable the hourly cron schedule (cancel it). Hourly sessions stay off until `RESUME BUG ESCAPE WORK` is received.
- `RESUME BUG ESCAPE WORK` — start a session immediately AND reinstate the hourly cron schedule if it was paused.
- `FREEZE AND ANALYZE` — (during an active work session) immediately cancel all triage-ci runs currently in progress, then analyze the logs from whatever ran before cancellation.

---

## Development Principles

### Verification runs: off-hours only
Real verification runs (non-mock) dispatch actual CI jobs and consume shared hardware. Only trigger real verification runs when:
- It is past 9 PM EST (02:00 UTC), **or**
- It is a weekend (Saturday or Sunday UTC)

During business hours, always use `mock-verify=true`. If unsure of the time, check before dispatching.

### Use Opus for hard tasks, Sonnet for easy ones
When spawning subagents or agent teams:
- *Opus*: analyzing complex logs to find root causes, writing or refactoring non-trivial bash/Python in the pipeline scripts, synthesizing findings across multiple files, any task where getting it wrong is costly
- *Sonnet*: fetching files, simple grep/search tasks, straightforward API calls, boilerplate edits with clear instructions

Opus costs ~5× more — don't use it for tasks Sonnet can handle reliably.

### Keep workflow runs under 25 minutes
Because logs are only readable after a job finishes, long runs mean long feedback cycles. Always aim for runs that complete in under 25 minutes (hard limit — container may reset at 30). Levers to pull:

- Restrict `test-workflows` input to a small subset (1-3 workflows) instead of all 29
- Shorten `lookback-days` (3-5 days) and `incremental-days` (2-3 days)
- Lower `MAX_CANDIDATES` to cap the number of candidates sent to the LLM
- Use `max-phase` to stop early (e.g. phase 2 only) when testing a specific phase
- Cancel the run at 30 minutes and analyze whatever partial logs were uploaded as artifacts
- Chunk LLM calls so each call returns quickly rather than one giant call timing out
- Modify the code directly — don't just tune inputs, fix the underlying bottleneck

If a run is going to exceed 25 minutes, cancel it, read the partial output, and iterate on a targeted fix before re-running. Changing the scripts themselves is always on the table.

---

## Seen-Candidate Cache (Deduplication Across Runs)

To avoid re-analyzing the same failures on every run, Phase 2 maintains a persistent cache of already-classified candidates.

*Mechanism:* `bug-escapes/state/seen.json` — a JSON file keyed by `workflow_basename|job_name|sorted_run_ids` with verdict values (`infra_noise`, `confirmed`, `no_logs`). Uploaded as a GitHub Actions artifact named `bug-escapes-seen-cache` (overwrite:true, 90-day retention) at the end of each run. Downloaded at the start of the next run via `_restore_seen_cache()` using `GITHUB_READ_TOKEN`.

*Cache key:* `workflow_basename|job_name|comma_sorted_run_ids` — identifies a specific regression streak. When a new run fails, the run IDs shift (e.g. `101,102,103` → `102,103,104`), producing a new key that bypasses the cache and gets processed fresh.

*Why artifact instead of git:* The detect job runs with `GITHUB_READ_TOKEN` (= `secrets.GITHUB_TOKEN` from tt-metal), which has no write access to `tt-auto-triage`. Git push fails silently. GitHub API artifact upload only needs the standard token.

*Benefits:*
- On stable workflows with no new failures: <1 min runtime, 0 LLM calls, 0 token cost
- New regression streaks (new run IDs) are always processed fresh
- 1.3KB artifact — negligible storage

*Known limitations:*
- Cache grows unboundedly — needs TTL eviction (TODO)
- Artifact expires after 90 days — next run after expiry does full re-scan (one wasted run)
- Race condition if two runs overlap: last upload wins, earlier run's new entries lost (rare)

---

## Known Issues / TODO

- [x] **Phase 2 chunking**: Fixed. Pre-extraction + chunking eliminates copilot timeouts. Validated: 88 candidates, 5 chunks of 20, all completed in ~2 min (32s/23s/24s/20s/17s). Commits: `9c82759d`, `70513c45`, `1c5ac90f` in tt-auto-triage.
- [x] **Verification job pruning**: Confirmed working. `build_pruned_yaml()` in `verify_common.sh` builds a single-entry test YAML; that YAML is pushed to the before/after branches via GitHub API before dispatch, so only the one failing job runs.
- [x] **Pruning should use an agent**: Replaced hardcoded Python `build_pruned_yaml()` with `_build_pruned_yaml_agent()` that calls the LLM API directly (same curl pattern as `check_failure_is_real`). Preserves ALL entry fields. Falls back to Python if agent unavailable. Python fallback also updated to not drop extra fields. Commit: `ba58e54f` in tt-auto-triage.
- [ ] **PRs ready to merge**: tt-auto-triage PR #1106 and tt-metal PR #42800 both exist as drafts. Not ready to merge until Phase 1 false-positive fix, cache TTL, and end-to-end validation with a real confirmed escape are done.
- [ ] **workflow-layers.json maintenance**: As tt-metal adds new workflows, the static config needs updating. Consider adding a Phase 1 fallback: if TEST_WORKFLOWS specifies a path that isn't in the config, auto-add it as 'other' with test_layer 'unknown' and log a warning rather than silently returning 0 workflows.
- [ ] **Phase 1 counting condition-skipped jobs as failures**: Jobs that are skipped due to `if:` condition evaluation (only `system.txt` present, no test output) are incorrectly counted as failures in Phase 1. These inflate candidate counts with jobs that never ran. Needs investigation.
- [ ] **Seen-cache TTL/eviction**: Cache grows unboundedly. Add a TTL (e.g. 30 days) so stale entries are pruned. Also mitigates artifact expiry (90-day GitHub limit) causing full re-scan.
- [x] **workflow-layers.json 2026-04-24 update**: Added 12 missing workflows. Commit `6e8b8520` in tt-auto-triage.
- [x] **End-to-end copilot test**: Full phase-4 run on `t3000-unit-tests.yaml` in progress (run 24899258967). `blackhole-post-commit.yaml` scan (run 24898787242) completed with 0 escapes — all 88 candidates were infra noise (hugepages/hardware failures).
- [x] **Phase 2 multi-run extraction**: Fixed false-negative where first failing run's log tail was dominated by post-test noise (AI summary tool errors, Docker cleanup), causing grep to find nothing and classifying the job as infra noise. Fix: try all 3 failing runs' logs for extraction, break only when error lines are found. Commit: `6b2ff7a7` in tt-auto-triage. Validation run 24899825890 completed (max-phase=2, t3000-unit-tests.yaml) — `t3k_ttnn_tests [wh_llmbox]` is currently passing in recent runs so false-negative can't be triggered; fix will activate on next episode of that job failing.
- [x] **Skip LLM for empty-snippet candidates (2026-04-26)**: After log download, if grep finds zero error lines across all tried runs, mark as `infra_noise` in cache and skip LLM entirely. Saves 57/88 LLM calls (65%) on blackhole-post-commit. Commit: `e2cc4e9` in tt-auto-triage.
- [x] **Persistent seen-candidate cache (2026-04-26)**: `bug-escapes/state/seen.json` keyed by `workflow_basename|job_name|sorted_run_ids`. Uploaded as GitHub Actions artifact `bug-escapes-seen-cache` at end of each run, downloaded at start of next. After first run populates cache, subsequent hourly runs take <1 min and spend 0 LLM tokens on stable workflows. Commits: `e2cc4e9`, `aa312a9` in tt-auto-triage. Validated: run 24945816274 loaded 88 entries, skipped all instantly.
- [x] **Reduce cron lookback from 14d → 2d (2026-04-26)**: Hourly cron now dispatches with `lookback-days=2`. Cuts candidate volume from 1000+ to ~88 per run. Runtime dropped from 102 min → ~18 min (with cache but before next fix).
- [x] **Lookback-window filter before job timeline fetches (2026-04-26)**: Previously fetched job timelines for up to 50 runs per workflow regardless of lookback window, causing 1750 API calls per run. Fixed by filtering `all_runs` to `created_at >= cutoff_date` after pagination and before spawning parallel job fetches. Commit: `e6231ad` in tt-auto-triage.
- [x] **Job-level noise index (2026-04-26)**: Persistently-empty jobs (empty snippet or no-logs) marked with `_noisy|wf|job` key in seen.json (6-hour TTL). Future runs skip log download entirely for those jobs. Blackhole-post-commit has ~42 noisy jobs; first run populates index, subsequent runs skip log downloads. Commit: `7fdc6ee` in tt-auto-triage. Validated: run 24958223978 completed in 4 min (first run), run 24958578768 completed in 3.5 min with max-phase=4 (full pipeline).
- [x] **Cron was dispatching max-phase=2 (2026-04-26)**: All hourly runs since at least 11:00 UTC dispatched with max-phase=2, causing phases 3 and 4 to never run. bug-escapes-output.json was never produced; aggregate job always showed "detection_failed_or_missing". Fixed by updating the nanoclaw cron task prompt to dispatch with max-phase=4. Validated: run 24958578768 produced a correct full-pipeline report with 0 escapes.

---

## Secrets / Credentials

- `tenstorrent/tt-metal` secret `AUTO_TRIAGE_TOKEN` = GitHub PAT used as `COPILOT_PAT` for copilot backend
- `CURSOR_API_KEY` = Cursor API key (existing secret in tt-metal)
- `GITHUB_TOKEN` = standard Actions token (for gh CLI, API calls)

---

## Key Files

```
tt-auto-triage / ebanerjee/bug-escapes
  bug-escapes/
    run.sh                      entry point, phase orchestration
    phase1_discover.sh
    phase2_candidates.sh        batch LLM classification — needs chunking fix
    phase3_fixpoints.sh
    phase4_classify.sh
    lib/
      cursor_agent.sh           LLM abstraction (cursor or copilot)
    verify/
      lib/verify_common.sh      check_failure_is_real() — dual backend
    .github/workflows/
      bug-escapes-ci.yaml       reusable workflow called from tt-metal
      verify-bug-escape-ci.yaml reusable verify workflow

tt-metal / ebanerjee/bug-escapes
  .github/workflows/
    triage-ci.yaml              trigger workflow, passes inputs + secrets
```
