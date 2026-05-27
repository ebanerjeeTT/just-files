# 1-Week Vertical Bug Escape Detection & Verification Plan

## A. Summary of Strategy

The core approach is **progressive scanning with prioritized verification**. Rather than attempting a single monolithic 60-day scan across all workflows (which would be API-expensive and fragile), we partition the work into **workflow cohorts** processed across the week. Non-Galaxy workflows are scanned first (Days 1-3) since they have no dispatch-time restrictions, while Galaxy workflows are scanned on Days 3-4 during their permitted midnight-4am windows. Each detection run targets a specific subset of workflows with the full 60-day lookback, using the dedup caches (`seen.json`, `seen_escapes.json`) to ensure no duplicate analysis across runs.

Verification is the bottleneck — each takes ~2 hours and concurrency is limited. The strategy is to **start verifying as soon as the first detection run completes** and run verifications continuously through the week. During working hours (9am-5pm EST): 1 at a time (4 verifications/day). Outside working hours: 2 at a time (up to 16/day). Theoretical max ~140 for the week; targeting **60-80 verified escapes**.

All results persist to `/workspace/group/bug-escapes-data/`. The dedup caches handle re-run safety; BrAIn's ledger handles cross-session persistence.

---

## B. Workflow Code Changes Required

### B1. Add `WORKFLOW_BATCH` env var to `run.sh`
Filter `pipeline-config.json` after Phase 1 to only include workflows matching a comma-separated list of basenames. Enables partitioned scanning without modifying `workflow-layers.json`.

### B2. Add early-exit on API rate limit in `phase2_candidates.sh`
After each GitHub API call, check `X-RateLimit-Remaining`. If remaining < 1000, write partial `consistent-failures.json` and exit with code 2 ("resume later"). `run.sh` treats exit 2 as non-fatal.

### B3. Cache commit details in `phase3_fixpoints.sh`
Cache commit + PR metadata in `state/commit-cache.json` keyed by SHA (7-day TTL). The same fix commit may appear across 10+ failures — caching avoids 20-30 redundant API calls per popular commit.

### B4. Add `--dry-run` mode to `verify/verify.sh`
When `DRY_RUN=1`, output what would be dispatched without actually dispatching. Allows pre-validating verification queues without burning CI resources.

### B5. Add artifact download retry in `bug-escapes-ci.yaml`
3 attempts, exponential backoff starting at 30s in the `aggregate` job. Prevents losing a full run's results to transient artifact download failures.

### B6. Extend dedup cache TTL to `LOOKBACK_DAYS`
Change both `seen.json` TTL (48h → 60d) and `seen_escapes.json` TTL (LOOKBACK_DAYS, which becomes 60d). **This is the biggest efficiency win** — subsequent runs skip everything already analyzed rather than re-doing the full 60-day scan.

### B7. Add `MAX_WORKFLOWS_PER_RUN` throttle to `phase2_candidates.sh`
Hard cap on workflows processed per run. Complements `WORKFLOW_BATCH` for API safety.

---

## C. Daily Schedule (EST)

### Monday (Day 1) — Setup + First Scan
```
06:00-09:00  Implement code changes B1-B7 on ebanerjee/bug-escapes
09:00-09:15  Init /workspace/group/bug-escapes-data/
09:15-12:00  Det-1: Unit/build/cpp workflows
             LOOKBACK_DAYS=60, LLM_BACKEND=copilot, CONSECUTIVE_RUNS=3
             ~2.5h, ~800 GH API calls
12:00-12:15  Save: Append Det-1 results to master ledger
12:15-17:00  Ver 1-4: Top-priority escapes (1 concurrent)
17:00-23:59  Ver 5-10: (2 concurrent)
23:00-23:30  Save: Persist verification results
```

### Tuesday (Day 2) — Second + Third Detection
```
00:00-04:00  Ver 11-14: Overnight (2 concurrent)
06:00-09:00  Det-2: Model/demo/ttnn workflows
             ~2.5h, ~1000 GH API calls
09:15-12:00  Ver 15-16 (1 concurrent)
12:00-15:00  Det-3: Perf/stress/t3000/tg/tgg workflows
             ~2.5h, ~1000 GH API calls
15:15-17:00  Ver 17 (1 concurrent)
17:00-23:59  Ver 18-23 (2 concurrent)
```

### Wednesday (Day 3) — Galaxy Detection
```
00:00-04:00  Det-4 (GALAXY): All 7 galaxy workflows
             LOOKBACK_DAYS=60, LLM_BACKEND=copilot, MAX_CANDIDATES=30
             ~2h, ~600 GH API calls
             NOTE: Detection can run anytime; only VERIFICATION dispatches need midnight-4am
04:15-09:00  Ver 24-28 (2 concurrent)
09:00-17:00  Ver 29-32 (1 concurrent)
17:00-23:59  Ver 33-38 (2 concurrent)
```

### Thursday (Day 4) — Galaxy Verification + Lower-threshold Re-scan
```
00:00-04:00  Galaxy-Ver 1-4: Dispatch Galaxy verifications (2 concurrent)
06:00-09:00  Det-5: Re-scan all non-Galaxy with CONSECUTIVE_RUNS=2, INCREMENTAL_DAYS=14
             Dedup cache skips everything seen in Det-1,2,3
             ~2h, ~500 GH API calls
09:15-17:00  Ver 39-42 (1 concurrent)
17:00-23:59  Ver 43-48 (2 concurrent)
```

### Friday (Day 5) — Focused Verification + Targeted Re-detection
```
00:00-04:00  Galaxy-Ver 5-8 (2 concurrent)
04:00-09:00  Ver 49-54 (2 concurrent)
09:00-12:00  Det-6: Top 3 highest-yield workflows, CONSECUTIVE_RUNS=2
             ~1.5h, ~400 GH API calls
12:15-17:00  Ver 55-57 (1 concurrent)
17:00-23:59  Ver 58-63 (2 concurrent)
23:00-23:30  Save: Mid-week progress report
```

### Saturday (Day 6) — Verification Marathon
```
00:00-04:00  Galaxy-Ver 9-12 (2 concurrent)
04:00-09:00  Ver 64-73 (2 concurrent)
09:00-17:00  Ver 74-77 (1 concurrent)
17:00-23:59  Ver 78-83 (2 concurrent)
            No detection runs today — clear the queue
```

### Sunday (Day 7) — Final Verifications + Report
```
00:00-04:00  Galaxy-Ver 13-16 (2 concurrent)
04:00-09:00  Ver 84-89 (2 concurrent)
09:00-12:00  Ver 90-93 (1 concurrent)
12:00-14:00  Det-7 (FINAL): All workflows, LOOKBACK_DAYS=60, INCREMENTAL_DAYS=3
             Catch anything new in the last 3 days. ~1h, ~300 GH API calls
14:00-17:00  Ver 94-96 (1 concurrent)
17:00-20:00  Generate final report
```

---

## D. GH API Budget Analysis

### Per-Detection-Run
```
Phase 1 (discover)            0 calls
Phase 2 per workflow:
  List runs                   1
  Fetch run details           50
  Download logs               15
  Subtotal                    66/workflow
Phase 2 (5 workflows)         330
Phase 3 per failure:
  Compare commits             1
  Commit details + PR + reviews  9
  Subtotal                    10/failure
Phase 3 (30 failures)         300
Phase 4 (classify)            0
TOTAL per run                 ~630 calls
```
With commit cache (B3), Phase 3 drops ~40% on subsequent runs.

### Per-Verification
```
Create branches               4
Read/modify test YAML         2
Dispatch workflows            2
Poll for completion           16 (2 workflows × 8 polls @ 120s over ~2h)
Download results              2
Cleanup branches              2
TOTAL                         ~28 calls
```

### Weekly Budget
```
Day         Detect    Verify    Total    Peak hr
Monday      800       120       920      ~400/hr
Tuesday     2000      260       2260     ~500/hr
Wednesday   600       300       900      ~350/hr
Thursday    500       280       780      ~300/hr
Friday      400       380       780      ~300/hr
Saturday    0         480       480      ~60/hr
Sunday      300       340       640      ~300/hr
TOTAL       ~4600     ~2160     ~6760
```
**Peak: ~500/hr on Tuesday = 3.3% of the 15,000/hr limit.**
**Weekly total: ~6,760 / 2,520,000 available = 0.27% utilization.**

---

## E. Data Persistence Design

### Directory Structure
```
/workspace/group/bug-escapes-data/
├── master-ledger.jsonl              # All detected escapes (append-only)
├── verification-results.jsonl       # All verification outcomes (append-only)
├── daily-summaries/
│   ├── 2026-05-11-monday.json
│   └── ...
├── detection-runs/
│   ├── det-1-monday-0915.json       # Raw output per detection run
│   └── ...
├── caches/                          # Backup copies of dedup caches
│   ├── seen.json
│   └── seen_escapes.json
└── final-report.json
```

### Master Ledger Schema (one JSON per line)
```json
{
  "escape_id": "sha256(fix_sha|test_name|wf_basename)[:12]",
  "detected_at": "2026-05-11T09:45:00-05:00",
  "detection_run": "det-1",
  "workflow": "unit-tests.yaml",
  "job_name": "unit-tests / test-dispatch-wh-b0",
  "test_name": "test_eltwise_binary",
  "test_layer": "ttnn",
  "fix_sha": "abc123def456",
  "fix_pr": 42567,
  "fix_pr_title": "Fix dispatch race in binary eltwise",
  "fix_layer": "tt-metalium",
  "fix_changed_files": ["tt_metal/impl/dispatch/command_queue.cpp"],
  "escape_direction": "vertical",
  "confidence": "high",
  "consecutive_failures": 5,
  "verification_status": "pending",
  "verification_id": null
}
```

### Deduplication
- `escape_id` (hash of fix_sha+test_name+workflow) checked against ledger before append
- `verification_status` flow: `pending` → `in_progress` → `verified` / `refuted` / `inconclusive`
- Pipeline's own caches handle detection-level dedup; BrAIn's ledger is the outer persistence layer

---

## F. Interruptibility Analysis

**Detection run killed mid-phase:**
- Phase 1: Stateless, free to re-run
- Phase 2: `seen.json` written incrementally per workflow; next run only re-processes current workflow
- Phase 3: Commit cache (B3) preserves fetched data; next run skips cached SHAs
- Phase 4: Pure local computation, completes in seconds, free to re-run

**Verification run killed mid-way:**
- Before dispatch: No resources consumed, re-run from scratch
- After dispatch: GitHub runs continue independently. Branch names are deterministic from fix SHA — BrAIn can find them via API and resume polling
- After completion: Results exist as GitHub artifacts. BrAIn re-downloads and saves

**BrAIn offline 12 hours:**
- In-flight CI workflows complete independently on GitHub (artifacts kept 90 days)
- On return: check for completed workflow runs, download artifacts, append to ledger
- Resume schedule from current slot
- Worst case: ~12-24 verifications lost from the queue, catchable via schedule slack

---

## G. Risk Factors and Mitigations

**Risk 1: Copilot rate limits or failures**
- Cap LLM calls per run via `MAX_CANDIDATES`
- Retry with exponential backoff (3 attempts: 30s/60s/120s)
- Schedule has built-in slack (Sunday is light)

**Risk 2: Verification returns ambiguous results (both before+after pass or fail)**
- Pre-validate with `DRY_RUN=1` before dispatch
- Prioritize high-confidence escapes first
- Mark inconclusive, retry only if Sunday slots available
- If ambiguous rate >30%, investigate test matrix pruning logic

**Risk 3: API budget surprise from external consumers**
- B2 provides hard safety valve (pause if remaining < 1000)
- Our peak is only 3.3% of limit — massive headroom
- BrAIn checks `GET /rate_limit` before each detection run; delays 30m if remaining < 5000

**Risk 4: Dedup cache corruption**
- BrAIn backs up caches after every detection run to `/bug-escapes-data/caches/`
- `bug-escapes-ci.yaml` provides a second artifact backup
- Full re-scan if cache lost: ~630 API calls per 5-workflow batch (acceptable)

**Risk 5: Galaxy verification window too narrow**
- 4 nights × 4 slots = 16 Galaxy verifications total (Thu-Sun)
- Prioritize by confidence score
- If >16 Galaxy escapes found, overflow to following week's Mon-Tue nights
