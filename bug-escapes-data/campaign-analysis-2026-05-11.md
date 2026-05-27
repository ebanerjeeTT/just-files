# Bug Escape Detection Campaign Analysis — 2026-05-11

## Executive Summary

**Run #575 (latest completed, 2026-05-11 15:13 UTC)** produced **0 bug escapes**.
The pipeline is working mechanically (no crashes, all phases complete), but it is
structurally unable to find escapes with the current configuration because:

1. A single noisy workflow (`blackhole-demo-tests.yaml`) saturates the MAX_CANDIDATES cap
2. That workflow's most recent run is still failing (queued/not yet completed), so the
   pre-filter cannot help — the candidates pass pre-filter (jobs DID have a recent success
   in the timeline), but Phase 3 finds them still-failing in the forward scan
3. All other workflows (35 of them) are never scanned because the cap is hit first

---

## Current State (Run #575)

```
Phase 2 output:   50 consistent failures (hit MAX_CANDIDATES=50 cap)
  Unique workflow:  1 (blackhole-demo-tests.yaml — ALL 50 candidates)
  Unique tests:    14 (37 are the same wan2.2 perf test across different jobs)
  Flaky:            4 (skipped by Phase 3)

Phase 3 output:
  Ongoing/still-failing: 46
  Skipped (flaky):        4
  Fix-points found:       0

Phase 4 output:
  Bug escapes: 0

Seen cache: 1,893 entries (915 infra_noise, 254 confirmed, 724 noisy)
```

### Workflow Scan Order Problem

Workflows are scanned in config order. `blackhole-demo-tests.yaml` is index [1] (second
after `blackhole-post-commit.yaml`). It has ~50 jobs, most of which are currently failing.
Once Phase 2 hits MAX_CANDIDATES=50, it `break 2`s out of both loops, and the remaining
33 workflows are never examined.

### Pre-Filter Effectiveness

The still-failing pre-filter (lines 330-354 of `phase2_candidates.sh`) **is working
correctly** but cannot help in this case. Here's why:

- The pre-filter checks if a job's *most recent run in the Phase 2 timeline* passed
- `blackhole-demo-tests.yaml` runs daily; the most recent run (May 11) is `queued` (not
  yet completed), so Phase 2's API fetch returns the May 10 run as the most recent with a
  conclusion
- Many of these jobs DO have a success somewhere in the 14-day timeline (just not the most
  recent), so they pass pre-filter but Phase 3's forward scan finds no fix transition

The pre-filter dropped **0** candidates in this run because the pattern is "fail, fail,
..., fail" with no recent success as the latest completed run.

### Why All Candidates Come From One Workflow

`blackhole-demo-tests.yaml` is a massive workflow with 50+ individual jobs running on
different hardware (bh_loudbox, bh_deskbox, bh_llmbox, bh_p300, bh_quietbox_2). Many of
these jobs share the same underlying test failures:

- **37 candidates**: Same wan2.2 performance test (`test_performance_wan.py`) failing
  across different jobs (different models using the same infra)
- **13 candidates**: Metal watchdog / dispatch timeout (hardware hang)

These are legitimate CI failures but they are NOT bug escapes — they're either performance
regressions at the model layer or hardware timeouts. A vertical bug escape requires a
fix in a LOWER layer (tt-metalium, tt-llk) for a test in a HIGHER layer.

---

## Blockers (Ranked by Impact)

### B1: Candidate Saturation from One Workflow (CRITICAL)

The MAX_CANDIDATES=50 cap is consumed entirely by `blackhole-demo-tests.yaml`. The 33
remaining workflows — including high-value targets like:
- `fast-dispatch-full-regressions-and-models.yaml` (layer=ttnn)
- `ttnn-run-sweeps.yaml` / `ttnn-stress-tests.yaml` (layer=ttnn)
- `t3000-integration-tests.yaml` (layer=models)
- `galaxy-unit-tests.yaml` (layer=tt-metalium)
- `blackhole-e2e-tests.yaml` (layer=multi-layer)

...are never scanned. These workflows have much better escape potential because they
cover TTNN and tt-metalium layers where cross-layer bugs actually live.

### B2: Test De-duplication Missing (HIGH)

37 of 50 candidates share the same `test_name` (wan2.2 perf test). The pipeline treats
each (workflow, job) pair as independent, but these are the same underlying failure
fanned out across hardware configs. Phase 2 should deduplicate by `(test_name,
failure_signature)` and pick one representative job per unique test failure.

### B3: Workflow Priority / Ordering (MEDIUM)

The pipeline scans workflows in config file order. High-escape-potential workflows
(TTNN sweeps, metalium unit tests, multi-layer e2e) should be scanned first. Pure
model-layer performance tests (blackhole-demo-tests, perf-models) should be scanned
last since model-layer-only failures are never vertical escapes.

### B4: Lookback Window Still Short for Fixed Bugs (LOW)

14-day lookback is reasonable for finding active failures, but many real escapes get
fixed within 1-3 days. With blackhole-demo-tests consuming all slots, even increasing
lookback won't help until B1 is solved.

---

## Recommended Next Steps

### Immediate (Before Det-2 on May 12 10:00 UTC)

1. **Per-workflow candidate cap**: Instead of a global `MAX_CANDIDATES=50`, use a
   per-workflow cap of ~5-10 candidates. This ensures every workflow gets scanned.
   Implementation: add a counter per workflow in the Phase 2 main loop and skip
   after hitting the per-wf cap.

2. **Prioritize escape-prone workflows**: Reorder `pipeline-config.json` to scan
   ttnn and metalium-layer workflows first, model-only performance tests last:
   ```
   Priority 1: fast-dispatch-*, ttnn-*, galaxy-unit-tests, umd-unit-tests
   Priority 2: t3000-*, blackhole-e2e-tests, models-t*-e2e
   Priority 3: *-demo-tests, perf-*, vllm-nightly
   ```

3. **Test dedup in Phase 2**: Before counting toward MAX_CANDIDATES, check if the
   same `(test_name, failure_signature)` has already been emitted. If so, skip.

### For Det-2 Through Det-7

```
Det-2 (Tue 10:00): Deploy per-wf cap + reordered config. lookback-days=14
Det-3 (Tue 16:00): Same config, verify new workflows are getting scanned
Det-4 (Wed 04:00): Extend lookback-days=30 if fix-points are found in Det-2/3
Det-5 (Thu 10:00): lookback-days=45, rescan all workflows with larger window
Det-7 (Sun 16:00): Final sweep with maximum lookback (60 days)
```

### Code Changes Needed (Priority Order)

1. **`phase2_candidates.sh`**: Add `MAX_CANDIDATES_PER_WORKFLOW` env var (default 10).
   After emitting 10 candidates from a workflow, `continue` to the next workflow.

2. **`phase2_candidates.sh`**: Add test-name deduplication. Track emitted
   `(test_name, failure_signature)` tuples; skip duplicates.

3. **`pipeline-config.json`**: Reorder workflows by escape potential (ttnn/metalium
   first, model-perf last).

4. **`phase3_fixpoints.sh`**: The `still_failing` check currently requires ALL
   subsequent runs to have the job present. Consider skipping gaps (already partially
   implemented with `consecutive_gaps`). Also consider looking at `conclusion == "success"`
   for ANY subsequent run, not just the immediate next one.

---

## High-Yield Workflow Targets

Based on layer coverage, these workflows are most likely to surface vertical escapes:

```
Workflow                                         Layer           Escape Potential
fast-dispatch-full-regressions-and-models.yaml   ttnn            HIGH (ttnn tests, metalium fixes)
ttnn-run-sweeps.yaml                             ttnn            HIGH
ttnn-stress-tests.yaml                           ttnn            HIGH
blackhole-e2e-tests.yaml                         multi-layer     HIGH (cross-layer tests)
t3000-fast-tests.yaml                            multi-layer     HIGH
galaxy-unit-tests.yaml                           tt-metalium     MEDIUM (metalium tests, llk fixes)
fast-dispatch-frequent-tests.yaml                tt-metalium     MEDIUM
metal-run-microbenchmarks.yaml                   tt-metalium     MEDIUM
t3000-integration-tests.yaml                     models          MEDIUM (integration = cross-layer)
tt-metal-l2-nightly.yaml                         multi-layer     MEDIUM (nightly = broader coverage)
```

These are the workflows that MUST be scanned before any model-only perf test workflow.

---

## Appendix: Run History

```
Run #  When              Candidates  Fix-points  Escapes  Notes
#571   ~May 8            145         crashed     0        Phase 3 crash (|| true fix)
#572   ~May 8            10          0           0        MAX_CANDIDATES=25, all still-failing
#575   May 11 15:13      50          0           0        MAX_CANDIDATES=50, pre-filter active
                                                          All from blackhole-demo-tests.yaml
```

Runs #567-570 were code iteration runs (May 8), not detection runs.

---

*Generated 2026-05-11 by BrAIn campaign analysis*
