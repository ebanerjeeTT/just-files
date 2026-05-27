# Bug Escapes — Findings Log

Running log of learnings, observations, and decisions from each work session.

---

## 2026-04-24 Session — Phase 2 False Negative + Efficiency Fix

### Status at session end (~20:35 UTC)

**Branch:** `ebanerjee/bug-escapes` in `tenstorrent/tt-auto-triage`
**Latest commit:** `8ddeda3` — both Phase 2 false-negative fixes applied and pushed
**Hourly cron:** `task-1777052684380-jiw31h` — updated to `lookback-days=2`, fires at :00 each hour
**Next run:** 22:00 UTC

---

### Fixes Applied This Session

**Bug 1 — `tail -c 30000 | grep` (commit `0500d49`)**
Pre-extraction grepped only the last 30KB of each log file. Fixed to `xargs grep -ih` (full file search).

**Bug 2 — Subdirectory matching blocks flat log files (commit `8ddeda3`, ROOT CAUSE)**
Phase 2 used `first_word` of job name to find a matching subdirectory (e.g. `ttnn-unit-tests.../`). Job subdirectories only contain `system.txt` — runner metadata ("Evaluating job.if"). Real test logs are flat numbered files at ZIP root: `0_JobName.txt`. Fixed: removed subdirectory matching, now greps `find "$run_log_dir" -type f -name "*.txt" ! -name "system.txt"`.

---

### Run Results

**Run 24905103884** (18:19 UTC, before Bug 2 fix): 88 candidates, 0 confirmed — subdirectory bug blocked real logs

**Run 24905897534** (18:38 UTC, with both fixes, `8ddeda3`): 102 min runtime, 882MB artifact (real logs downloaded), still **0 confirmed**

**Why 0 confirmed is probably accurate:**
- Recent "failing" runs for ttnn-unit-tests are actually SKIPPED jobs (condition eval only → just `system.txt`, no test output). Phase 1 may be counting job-skips as failures.
- Older runs with real test failures have expired logs (old run IDs 16xxx-20xxx → 410 Gone from GitHub API).
- The 14-day lookback pulls in hundreds of stale candidates the LLM correctly dismisses.

---

### Efficiency Fix

**Problem:** `lookback-days=14` → 1000+ candidates across 35 workflows → 102 min runtime → same stale candidates re-processed every hour.

**Fix:** Updated cron dispatch to `lookback-days=2`. Hourly runs examine only failures that started within last 2 days. Expected runtime: 5-15 min.

---

### What To Watch At 22:00 UTC

1. Runtime — should be <15 min (was 102 min)
2. Candidate count — should be low (was 1000+ across all workflows)
3. Any CONFIRMED failures — check pre-extracted snippets for real test output
4. Artifact size — should be much smaller than 882MB

---

### Outstanding TODOs

1. **[NEXT]** Watch 22:00 UTC run — validate `lookback-days=2` efficiency fix
2. **[MEDIUM]** Investigate Phase 1 counting skipped jobs as failures (condition-eval-only skips shouldn't be candidates)
3. **[MEDIUM]** Persistent dedup registry — skip `(workflow, job_name, streak_start_run_id)` already processed in prior runs
4. **[LOW]** Open PR for `ebanerjee/bug-escapes` changes into tt-auto-triage main
5. **[BLOCKED]** tt-metal PR #42800 — waiting on tt-auto-triage PR first

---

## 2026-04-24 — Agent-based pruned YAML generation

*Root cause of brittle pruning:* `build_pruned_yaml()` in `verify_common.sh` used a hardcoded Python extractor that only preserved `name, cmd, skus, owner_id, team` — silently dropping any extra fields like `model-name`, `docker_image`, or custom runner config. As workflows evolve, this can cause pruned YAMLs to be missing required runner hints.

*Fix: agent-based generation with Python fallback:* Added `_build_pruned_yaml_agent()` that calls the LLM API directly (same curl approach as `check_failure_is_real`, branches on `LLM_BACKEND`). The agent receives the full entry JSON + test name + job name and produces a correct pruned YAML preserving ALL fields. If the LLM is unavailable, the response is malformed, or YAML validation fails, it falls back to the Python extractor. Python fallback also updated to preserve extra fields via `for k,v in entry.items(): if k not in pruned: pruned[k] = v`. Commit: `ba58e54f` in tt-auto-triage.

*Validation run 24901845331:* Full phase-4 detection run on `t3000-unit-tests.yaml` (copilot backend, mock-verify=true, auto-verify=true, lookback=7d). Completed in ~4 min. 47 candidates → 0 confirmed escapes → `verify-all` skipped (no qualifying escapes). The new agent-based pruning path was NOT exercised (expected — no escapes to verify). Code is deployed; path will activate on first real escape with a valid fix commit.

*workflow-layers.json gap found:* Scanning `models-post-commit.yaml` produced 0 workflows in Phase 1 — the workflow wasn't in the static config. Root cause: 12 workflows added to tt-metal since the config was last updated weren't registered. Added `models-post-commit`, `ttnn-post-commit`, `cpp-post-commit`, `ops-post-commit`, `tt-train-post-commit`, `blackhole-e2e-tests`, `models-t1/t2/t3-e2e-tests`, `runtime-perf-tests`, `galaxy-e2e-tests`, `galaxy-perf-tests`. Commit: `6e8b8520` in tt-auto-triage. Validated: run 24902236618 confirms `models-post-commit.yaml` now found (Phase 1: 1 workflow, 1 'other').

---

## 2026-04-24 — Initial sessions

*Smoke test fix:* `run.sh` Phase 0 unconditionally checked `agent --version` even when `LLM_BACKEND=copilot`. Fixed to branch on `LLM_BACKEND` and check `copilot --version` instead. Commit: `f7b5e5a7` in tt-auto-triage.

*Phase 2 timeout:* Copilot backend times out at 300s when sent 80-100+ candidates in a single LLM call. Cursor handles large batches faster. Batches of ~7 candidates work fine. Fix needed: chunk Phase 2 into ~20 candidates per call.

*Deduplication confirmed working:* Run 24855762867 correctly grouped 3 "board reset" vLLM jobs into 1 group, suppressing 2 duplicates.

*Log tail reading:* Errors in long logs appear near the end, not the beginning. Fixed `detect_failures.py` to read the last 100k chars instead of the first. Confirmed this fixed T3K Llama signature extraction.

*Verification pruning confirmed:* `build_pruned_yaml()` correctly builds a single-entry test YAML and pushes it to before/after branches. Only the failing job runs during verification.

*triage-ci.yaml (tt-metal):* Copilot credentials wired up — `llm-backend` input and `COPILOT_PAT: ${{ secrets.AUTO_TRIAGE_TOKEN }}` added to both detect and verify jobs. Committed directly (was pending from prior session).

*End-to-end copilot run:* Has not completed successfully yet. Blocked on Phase 2 batch size issue.

---

## 2026-04-24 — Phase 2 pre-extraction fix + Phase 1 filter fix

*Phase 1 basename filter:* `TEST_WORKFLOWS=blackhole-post-commit.yaml` was matching against full paths like `.github/workflows/blackhole-post-commit.yaml`. Fixed Phase 1 filter to use `endswith("/"+wf)` so bare filenames work. Commit: `4b05fb95` in tt-auto-triage.

*Phase 2 pre-extraction:* Root cause of copilot 300s timeouts: LLM was doing 60+ file I/O operations (list dir, grep, read) per 20-candidate chunk. Fixed by pre-extracting error lines in bash before the LLM call. Each candidate now includes grep output directly in the prompt instead of a log directory path. Commits: `70513c45` (phase2), `1c5ac90f` (prompt) in tt-auto-triage.

*Validation run 24898787242:* 88 candidates, 5 chunks of 20, all completed without timeout. Chunk times: 32s, 23s, 24s, 20s, 17s. Total Phase 2: ~2 min. 0 confirmed escapes (blackhole-post-commit may be healthy, or all 88 were infra noise — need broader scan).

*Backend log labels:* `cursor_agent_query:` messages now say `copilot_agent:` / `cursor_agent:` based on actual backend. Commit: `a9d10cdd` in tt-auto-triage.

---

## 2026-04-24 — First full phase-4 copilot run + false-negative root cause

*run.sh API key check:* `run.sh` was unconditionally requiring `CURSOR_API_KEY` even on copilot backend. Fixed to guard with `[ "${LLM_BACKEND:-cursor}" != "copilot" ]`. Commit: `352ce0f6` in tt-auto-triage.

*End-to-end copilot run succeeded:* Run 24899258967 completed full Phase 0→4 on `t3000-unit-tests.yaml` with copilot backend in ~4 minutes. Phase 2: 47 candidates, 17 likely_flaky, 3 chunks, 1 CONFIRMED.

*Confirmed ongoing failure:* `AccessorTests/AccessorBenchmarks.PagesIteratorInterleaved/3` (job: `download-artifacts`, `t3000-unit-tests.yaml`) confirmed real test failure with high confidence. Error: `TT_THROW @ system_memory_manager.cpp:717`. Phase 3 found no fix point — still active on main.

*Phase 4 correctly produced 0 bug escapes:* Ongoing failures with no fix commit are not bug escapes. Written to `ongoing-failures.json` instead. `bug-escapes-output.json` correctly shows empty array.

*Phase 2 false-negative root cause found:* Phase 2 downloaded only the FIRST failing run's logs for pre-extraction, then broke out of the loop. When that run's log tail was dominated by post-test noise (AI summary tool errors, Docker cleanup output), the grep found nothing and the job was classified as infra noise — even though later runs had real FAILED lines. Affected: `t3k_ttnn_tests [wh_llmbox]` and multiple other jobs. The same AccessorBenchmarks failures that were caught in `download-artifacts` were also present in `t3k_ttnn_tests [wh_llmbox]` but missed.

*Phase 2 multi-run extraction fix:* Changed download loop to try all 3 failing runs' logs for extraction, breaking only when error lines are found. If all runs' tails are empty, falls back to "no errors." Commit: `6b2ff7a7` in tt-auto-triage. Validation run 24899825890 in progress (max-phase=2, t3000-unit-tests.yaml).

*Artifact naming:* Both `bug-escapes-output` (26MB, full detection output) and `bug-escapes-final-report` (1KB, final report) are uploaded with different names — no conflict. Earlier diagnosis of artifact overwrite was incorrect.

---

---
## 2026-04-24 18:08 UTC — blackhole-post-commit.yaml

- **Run**: https://github.com/tenstorrent/tt-metal/actions/runs/24904276503
- **Branch**: ebanerjee/bug-escapes
- **Workflow scanned**: `.github/workflows/blackhole-post-commit.yaml`
- **mock-verify**: true (business hours)
- **Lookback**: 7 days, 3 consecutive runs threshold
- **Candidates**: 88 (2 marked likely_flaky)
- **Confirmed consistent failures**: 0
- **Bug escapes**: 0 (horizontal=0, vertical=0, cross_layer=0, unknown=0)
- **Verdict**: Clean — no bug escapes detected in blackhole post-commit over last 7 days.

---
## 2026-04-24 18:19 UTC — blackhole-post-commit.yaml (RESUME session)

- **Run**: https://github.com/tenstorrent/tt-metal/actions/runs/24905103884
- **Branch**: ebanerjee/bug-escapes @ 4de42be (expanded grep patterns)
- **Workflow scanned**: `.github/workflows/blackhole-post-commit.yaml`
- **Status**: In progress as of 18:24 UTC

### Investigation: Phase 2 false negative root cause

Traced false negatives from run 24904276503 (all 88 candidates classified infra noise despite `logs/run_24597143912/0_ttnn-unit-tests (P150b) _ ttnn fused group 1.txt` (2.5MB) containing real TT_FATAL + AssertionError + FAILED lines).

**Root cause**: Pre-extraction logic uses `tail -c 30000 {} | grep` — this only greps the last 30KB of each log file. For a 2.5MB pytest log where the test failure summary appears ~1.5MB into the file (followed by teardown/cleanup output), the failure lines are NOT in the last 30KB and are therefore missed.

**Fix**: Changed `xargs -I{} tail -c 30000 {} | grep` → `xargs grep -ih`. Grep now searches the full file content, with `tail -40` still capping output. Commit: `0500d49` pushed to `ebanerjee/bug-escapes`.

**Validated locally**: Test showed `tail -c 30000 | grep` returns nothing for a file where failure appears at byte ~150K of a 184KB file; `grep` directly finds it.


## 2026-04-24 19:57 UTC — fast-dispatch-frequent-tests.yaml

- **Run ID**: [24909070354](https://github.com/tenstorrent/tt-metal/actions/runs/24909070354)
- **Workflow scanned**: `.github/workflows/fast-dispatch-frequent-tests.yaml`
- **Backend**: copilot | **mock-verify**: true | **lookback**: 7d | **threshold**: 3 consecutive
- **Phase 2**: 14 candidates reviewed by LLM → 0 confirmed consistent failures
- **Bug escapes**: 0 (horizontal=0, vertical=0, cross_layer=0, unknown=0)
- **Verify**: skipped (no qualifying escapes)
- **Duration**: ~1 minute
- **Notes**: Clean scan. fast-dispatch-frequent-tests appears healthy over the last 7 days — no sustained multi-run failures survived LLM triage.


## 2026-04-24 20:05 UTC — blackhole-e2e-tests.yaml

- **Run ID**: [24909240592](https://github.com/tenstorrent/tt-metal/actions/runs/24909240592)
- **Workflow scanned**: `.github/workflows/blackhole-e2e-tests.yaml`
- **Backend**: copilot | **mock-verify**: false | **lookback**: 7d | **threshold**: 3 consecutive
- **Phase 2**: 85 candidates reviewed by LLM (5 chunks of 20) → 0 confirmed consistent failures
- **Bug escapes**: 0 (horizontal=0, vertical=0, cross_layer=0, unknown=0)
- **Verify**: skipped (no qualifying escapes)
- **Duration**: ~5 minutes
- **Notes**: Large candidate pool (85) for blackhole-e2e — significantly more than other workflows. All classified as infra noise or flaky after LLM review. No real bug escapes detected in last 7 days on this workflow.


## 2026-04-24 21:08 UTC — single-card-demo-tests.yaml

- **Run ID**: [24911646290](https://github.com/tenstorrent/tt-metal/actions/runs/24911646290)
- **Workflow scanned**: `.github/workflows/single-card-demo-tests.yaml`
- **Backend**: copilot | **mock-verify**: false | **lookback**: 7d | **threshold**: 3 consecutive
- **Phase 2**: 66 candidates reviewed by LLM (4 chunks of 20, 1 flaky-filtered) → 0 confirmed consistent failures
- **Bug escapes**: 0 (horizontal=0, vertical=0, cross_layer=0, unknown=0)
- **Verify**: skipped (no qualifying escapes)
- **Duration**: ~8 minutes
- **Notes**: 66 candidates is a notable candidate volume for single-card-demo-tests, though LLM classified all as infra noise or flaky. The 1 flaky-filtered candidate is expected. No real bug escapes detected in last 7 days on this workflow.


---
## 2026-04-26 Session — Seen-cache + Empty-snippet optimizations

### Changes committed: `e2cc4e9` on `ebanerjee/bug-escapes`

**Change 1 — Skip LLM for empty-snippet candidates**
- After log download, if grep finds zero error lines in all tried runs, mark as `infra_noise` in cache and skip LLM.
- Old behavior: send `[no lines matched...]` placeholder to LLM → LLM says "infra noise" → pure token waste.

**Change 2 — Persistent seen-candidate cache**
- `bug-escapes/state/seen.json` keyed by `workflow_basename|job_name|sorted_run_ids`
- Verdicts: `no_logs`, `infra_noise`, `confirmed`
- Flow: pull remote cache at start (remote wins) → skip seen candidates → pre-mark chunk as `infra_noise` → upgrade to `confirmed` if LLM confirms → push updated cache at end with `[skip ci]`
- Effect: on next hourly run, all candidates already classified are skipped entirely (no API calls, no LLM calls)

### Validation run: 24945612804
- Single workflow scan: `blackhole-post-commit.yaml`
- Lookback: 2 days, max-phase=2
- Expected: fast runtime, seen cache populated on first run (base case); second run should skip all cached candidates

### Outstanding TODOs
1. **[NEXT]** Check run 24945612804 result — verify cache was written and pushed
2. **[MEDIUM]** Phase 1 counting condition-skipped jobs as failures — investigate
3. **[MEDIUM]** Open PR for ebanerjee/bug-escapes → tt-auto-triage main
4. **[BLOCKED]** tt-metal PR #42800 — waiting on tt-auto-triage PR

### Cache mechanism — revision: artifact-based (commit `aa312a9`)

**Problem with git push approach**: The detect job runs with `GITHUB_READ_TOKEN` (= `secrets.GITHUB_TOKEN` from tt-metal), which has no write access to `tt-auto-triage`. Git push fails silently.

**Fix**: Artifact-based rolling cache
- End of each run: `bug-escapes/state/seen.json` is uploaded as artifact `bug-escapes-seen-cache` (overwrite:true, 90-day retention) via new workflow step in `bug-escapes-ci.yaml`
- Start of next run: `_restore_seen_cache()` downloads artifact via GitHub API using existing `GITHUB_READ_TOKEN` (read access to tt-metal artifacts is fine)

**Validation run 2**: `24945727621` — dispatched at 01:55 UTC to verify artifact upload + correct cache restoration on next run

### Run 3 result — cache hit confirmed ✅

- *Run ID*: 24945816274
- *Seen cache loaded*: 88 entries from artifact (1.3KB)
- *All 88 candidates*: skipped immediately ("already classified — skipping (cached)")
- *LLM calls*: 0
- *Log downloads*: 0 (script exited early: "No logs downloaded — skipping workflow")
- *Runtime*: <1 minute (vs 2.5 min first run, 102 min original)

### Summary of efficiency gains

| Scenario | Runtime | LLM calls | Token cost |
|---|---|---|---|
| Original (lookback=14d, no cache) | 102 min | ~100 | very high |
| First run after fix (lookback=2d) | ~2.5 min | 31 (57 skipped by snippet filter) | low |
| Subsequent runs (cache populated) | <1 min | 0 | none |

New candidates (new regression streaks with new run IDs) will still be processed normally.

### Outstanding TODOs
1. **[NEXT]** Open PR for ebanerjee/bug-escapes → tt-auto-triage main
2. **[MEDIUM]** Phase 1 counting condition-skipped jobs as failures — investigate
3. **[BLOCKED]** tt-metal PR #42800 — waiting on tt-auto-triage PR

---
## 2026-04-26 — Lookback-window filter for job timeline API calls

*Root cause of 18-min hourly runtime (post-cache):* Even with the seen-candidate cache skipping LLM calls, Phase 2 still fetched up to 50 runs per workflow for the job timeline phase. For blackhole-post-commit (~6 runs/day), 50 runs = ~8 days of data. Each run required 1 API call, so 50 runs × 35 workflows = 1750 API calls per hourly run. This was the dominant time cost.

*Fix:* After the pagination loop, filter `all_runs` to `created_at >= cutoff_date` before spawning job timeline fetches. For lookback=2d, blackhole-post-commit drops from 50 → ~12 runs; infrequent workflows from 50 → 3-5. Commit: `e6231ad` in tt-auto-triage.

*Expected improvement:* ~70% reduction in job timeline API calls → estimated runtime drop from ~18 min to ~5 min.

*Validation run:* 24957904213 — completed, 14 min. Still slow because the noisy-job index wasn't yet in place.

---
## 2026-04-26 — Job-level noise index + cron max-phase bug

### Job-level noise index (commit `7fdc6ee`)

*Problem:* After the lookback-window filter, runtime was still 14 min because ~42 jobs in single-card-demo-tests.yaml and similar workflows persistently produce empty log snippets. The exact-key cache only skips them after seeing new run IDs — but every hourly run has new run IDs for actively-running workflows, so they keep going through the log-download + grep loop and finding nothing.

*Fix:* Two new helpers in phase2_candidates.sh:
- `_mark_job_noisy()` — writes `_noisy|wf_basename|job_name → ISO_timestamp` into seen.json
- `_is_job_noisy()` — returns true if that key exists and is < 6 hours old

Set at: no-logs path and empty-snippet path (NOT LLM-classified infra_noise — those had real content). Checked before log download: if noisy, mark the run as `infra_noise` and skip entirely.

*Validation run 24958223978:* 4 minutes (first run — populated 51 noisy entries from single-card-demo-tests.yaml and a few others). Subsequent runs expected ~2 min.

### Cron was using max-phase=2 (bug found and fixed)

*Discovery:* All hourly cron runs since at least 11:00 UTC were using `max-phase=2`. This caused the pipeline to stop after Phase 2 — no fix-point analysis (Phase 3), no classification (Phase 4), no `bug-escapes-output.json`. The aggregate job always showed "detection_failed_or_missing".

*Root cause:* The nanoclaw cron task prompt was dispatching with `max-phase=2`, which was a development/testing parameter that was never updated to `max-phase=4` for production.

*Fix:* Updated nanoclaw cron task prompt to dispatch with `max-phase=4`.

*Validation run 24958578768:* Full phase 0→4 in 3.5 minutes. `bug-escapes-output.json` produced with proper structure, 0 escapes, aggregate report correct.

### Runtime progression summary

```
Original (lookback=14d, no cache):        102 min
After lookback=2d cron update:             18 min  (still slow from API calls)
After lookback-window filter (e6231ad):    14 min  (still slow from empty-log downloads)
After job noise index, run 1 (7fdc6ee):    4 min   (first run, populating noisy index)
After job noise index, full phase 4:       3.5 min (warm noisy index, all 4 phases)
Target:                                    <3 min warm
```

### Current seen.json state (as of run 24958578768)
- Total entries: 3127
- Exact-key: 3075 (3050 infra_noise, 25 no_logs, 0 confirmed)
- Noisy-index entries: 52 (51 single-card-demo-tests.yaml, 1 models-t1-e2e-tests.yaml)

### Outstanding TODOs
1. **[MEDIUM]** Cache TTL/eviction — 3127 entries, growing unboundedly with each hourly run
2. **[MEDIUM]** Phase 1 condition-skipped jobs — jobs with `if:` condition failures show as candidates but never ran
3. **[MEDIUM]** PRs #1106 (tt-auto-triage) and #42800 (tt-metal) — drafts, not ready to merge

---
## 2026-04-29 Session — Phase 2 bash bugs + Phase 3 silent crashes + first full pipeline run

### Branch state
- **Branch**: `ebanerjee/bug-escapes` in `tenstorrent/tt-auto-triage`
- **Worktree**: `/workspace/group/worktrees/bug-escapes-dispatch-fix/`
- **Final commit this session**: `48f365b`
- **Hourly cron**: `task-1777052684380-jiw31h` — active, fires at :00 each hour, dispatches `max-phase=4`

### Bug fixes applied

**Bug 1 — `local` keyword outside function (commit `3d86115`)**
`local _snippet_for_prompt=...` at script top-level in `phase2_candidates.sh`. Removed `local` keyword.

**Bug 2 — Multi-doc agent output corrupting jq `length` arithmetic (commit `3837610`)**
`cursor_agent.sh` emitted multiple JSON documents; `jq 'length'` output `7\n7`; `$(( 7\n7 - 1 ))` is bash arithmetic syntax error. Fixed: `jq -c '.' | head -1 | jq '.'` in cursor_agent.sh; `| head -1` + `${num_results:-0}` fallback in phase2.

**Bug 3 — Phase 3 unguarded jq under `set -euo pipefail` (commit `b7555ec`)**
Multiple bare `jq` calls in `phase3_fixpoints.sh` exited non-zero, silently crashing the script. Intermittent crash at candidate [15/18]. Fixed: ERR trap for line-number diagnostics + `2>/dev/null || echo <default>` guards on all bare jq calls.

**Bug 4 — `.jobs[]` on API fallback `{}` causes jq exit 5 (commit `48f365b`)**
`get_jobs_for_run` falls back to `{}` on API failure. `.jobs[]` on `{}` → jq exits 5 (null not iterable). `set -o pipefail` propagates through `| head -1` despite `2>/dev/null`. Crash at line 136 (`job_conclusion` pipeline). Fixed: `(.jobs // [])[]` in both `job_conclusion` and `pf_conclusion` queries + `|| echo ""` on subshell.

### First successful full pipeline run (max-phase=4)

**Run 25090346606** — completed 04:54 UTC Apr 29, all 4 phases successful.
- `consistent-failures.json`: 33 entries
- `fix-points.json`: 16 entries
- `ongoing-failures.json`: 17 entries
- **6 bug escapes found** (all `type=vertical`, `medium`/`low` confidence)

```
test_flash_mla_decode              tt-metal-l2-nightly.yaml    fix: 131f0fafac
test_sdpa_prefill                  tt-metal-l2-nightly.yaml    fix: b0c9e59f6e
test_binary_uint16                 blackhole-post-commit.yaml  fix: 9f659be2c4
[BH-LB] Llama-3.1-8B-Instruct     vllm-nightly-tests.yaml     fix: b10dc75356
[WH-N300] NoOp test model (vLLM)   vllm-nightly-tests.yaml     fix: 831bbb9656
[WH-T3K] Qwen3-VL-32B-Instruct    vllm-nightly-tests.yaml     fix: b10dc75356
```

None pass `type=vertical AND fix_confidence=high` → auto-verify never triggered. Calibration issue.

### Hourly run reliability (Apr 29)
3 crashes pre-fix (08:01, 10:01, 19:00 UTC). All post-fix runs (16:01 onward) succeeded except 19:00 which was bug 4 (fixed in 48f365b). 20:01–22:02 all success.

### Readiness assessment
- Phases 0-2: stable
- Phase 3: ~95% stable after bug 4 fix
- Phase 4: stable
- Auto-verify: never triggered (confidence calibration gap)
- Overall: ~70-80% to fully autonomous. Safe for nightly with `auto-verify=false`.

### Outstanding TODOs
1. **[HIGH]** Tune `find_fix_commit.txt` prompt — all current escapes are medium/low confidence, blocking auto-verify
2. **[MEDIUM]** Monitor phase 3 for further crash sites — ERR trap now pinpoints them
3. **[MEDIUM]** Segformer shape regression — will re-surface when `_noisy` cache TTL expires
4. **[MEDIUM]** PRs #1106 (tt-auto-triage) and #42800 (tt-metal) — still drafts
5. **[LOW]** seen.json cache TTL/eviction — still growing unboundedly

---
## 2026-04-30 (late Apr 29 UTC) — Auto-verify threshold + cron update

### Changes

**commit `774f2f0` — lower auto-verify threshold from `high` to `medium`**

Root cause of auto-verify never triggering: `bug-escapes-ci.yaml` filter step and `aggregate_report.sh` both required `fix_confidence == "high"`. All 6 current escapes are `medium`. The confidence score is a cost-control pre-filter; the verification step itself is the correctness gate (wrong attributions return "refuted"). Lowering to medium is safe — `verify-budget-hours` caps total CI spend.

Example justifying the change: fix `b0c9e59f6e` "SDPA: make lightweight mask gate compile-time" for `test_sdpa_prefill` — SDPA is the exact subsystem, clearly the right commit, but rated medium because the prompt requires the exact test function name for "high".

Files changed:
- `bug-escapes-ci.yaml`: filter now selects `.fix_confidence == "high" or .fix_confidence == "medium"`
- `aggregate_report.sh`: `was_verification_candidate`, `missing_artifact`, and `not_attempted` status checks all updated to match

Low-confidence escapes still excluded.

**nanoclaw task `task-1777052684380-jiw31h` updated — `auto-verify=true`**

Cron now dispatches with `auto-verify=true`. Each hourly run will both detect escapes AND dispatch verification CI runs (before/after fix commit) for qualifying medium/high confidence vertical escapes. Budget: 6 hours per run (`verify-budget-hours` default).

### Outstanding TODOs (updated)
1. **[WATCH]** Monitor first cron run with `auto-verify=true` — verification has never been exercised end-to-end in production
2. **[MEDIUM]** Monitor phase 3 for further crash sites — ERR trap pinpoints them fast
3. **[MEDIUM]** Segformer shape regression — will re-surface when `_noisy` cache TTL expires
4. **[MEDIUM]** PRs #1106 (tt-auto-triage) and #42800 (tt-metal) — still drafts
5. **[LOW]** seen.json cache TTL/eviction — still growing unboundedly

---
## 2026-04-30 04:10 UTC — Auto-verify confirmed working; 0 escapes is correct

### Observation

Runs 00:00, 02:00, 03:00 UTC all completed successfully with `auto-verify=true` confirmed passed in logs (03:00 run: `auto-verify: true` in job setup). BUT `verify-all: skipped` on all three. This is NOT a bug.

Root cause of 0 escapes: the 5 consistent failures (t3k perf models, Fabric Collectives) are **still failing** — no fixpoints have appeared. Bug escapes require a fixpoint (test was failing, then started passing on/after a commit). Without fixpoints → 0 bug escapes → nothing to verify. The pipeline is correct.

03:00 run artifact confirms:
- consistent-failures.json: 5 entries (t3k CNN, Falcon 40B, Falcon 7B, Gemma3 ViT, Fabric Collectives ubench)
- fix-points.json: 0 entries
- bug-escapes-output.json: 0 bug escapes

The original 6 escapes from the Apr 29 first run are captured in seen.json — they were regressions that had already been fixed. The 5 current failures are new ongoing regressions not yet fixed.

### Auto-verify readiness: confirmed

- `auto-verify=true` is correctly propagated from nanoclaw → triage-ci.yaml → bug-escapes-ci.yaml
- Filter step executes and reports qualifying escape count
- verify-all correctly triggers when qualifying escapes > 0 (pending real test data)
- End-to-end verified as correct for the 0-escape case

### Pipeline reliability Apr 29-30

| Run time | Status | Escapes |
|----------|--------|---------|
| 00:00 | success | 0 |
| 01:00 | cancelled (Cursor API flaky) | - |
| 02:00 | success | 0 |
| 03:00 | success | 0 |
| 04:00 | in progress | TBD |

(01:00 cancellation is transient — Cursor API recovered by 02:00)

### Outstanding TODOs (updated)
1. **[WATCH]** Wait for a run that actually finds bug escapes to confirm verify-all triggers end-to-end
2. **[MEDIUM]** Circuit-breaker for Cursor API flakiness — if N consecutive agent calls fail, skip remaining to avoid 90-min hangs
3. **[MEDIUM]** seen.json cache TTL/eviction — still growing unboundedly
4. **[LOW]** PRs #1106 (tt-auto-triage) and #42800 (tt-metal) — still drafts

---
## 2026-05-01 — Phase 4 ARG_MAX crash (commit `a6bde85`)

### What happened
07:00 UTC run failed. Phase 3 found 62 fix points (much larger than previous runs due to expanded lookback data). Phase 4 crashed at line 262: `/usr/bin/jq: Argument list too long` (exit 126).

Root cause: `jq -n --argjson escapes "$bug_escapes"` passes the full accumulated array as a shell argument. With 62 entries each having analysis text, URLs, file lists etc., `$bug_escapes` exceeded the OS ARG_MAX limit.

Fix: write `$bug_escapes` to a temp file, use `--slurpfile escapes "$_escapes_tmp"` instead. `--slurpfile` wraps in an outer array, so reference as `$escapes[0]`. Committed `a6bde85`, pushed.

### Cron status
All runs Apr 30 15:00 UTC through May 1 06:00 UTC: success, 0 escapes. 07:00 UTC: failure (this bug). All subsequent runs (08:00 onward) should succeed with the fix.

### Note on 62 fix points
This is significant — earlier runs found 0-16 fix points. 62 suggests more regressions have resolved in the lookback window. Once Phase 4 runs successfully, could find many bug escapes worth verifying.

### Outstanding TODOs (updated)
1. **[WATCH]** Monitor next run to confirm Phase 4 no longer crashes with 62+ fix points
2. **[WATCH]** Check if 62 fix points produce qualifying medium/high vertical escapes → verify-all should trigger
3. **[MEDIUM]** Circuit-breaker for Cursor API flakiness in Phase 3
4. **[MEDIUM]** seen.json cache TTL/eviction — growing unboundedly
5. **[LOW]** PRs #1106 (tt-auto-triage) and #42800 (tt-metal) — still drafts
