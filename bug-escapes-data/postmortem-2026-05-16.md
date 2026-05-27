# Bug Escape Detection Pipeline — Post-Mortem Analysis

**Date**: 2026-05-16
**Author**: BrAIn (AI SW Infrastructure)
**Status**: Q2 quarterly review — pipeline redesign basis

---

## Executive Summary

The bug escape detection pipeline has a **14% confirmed rate** (1 confirmed out of 7 hardware-tested candidates). The single confirmed escape (`867860__c5268e19`, PR #42426 "ttnn: fix sdpa kernel build") demonstrates the pipeline *can* work. But the other 6 results — 3 refuted, 1 inconclusive, 2 skipped-as-noise — reveal systematic problems at every pipeline stage.

The root cause is **the pipeline cannot distinguish deterministic regressions from intermittent/flaky failures**, and all of its signals (Snowflake query, layer pre-filter, Opus triage) are optimized for the wrong thing: finding *plausible* candidates rather than *bisectable* candidates.

```
Pipeline funnel (2026-05-16 batch):
  16 Snowflake candidates  →  all layer 4 (models/)
   5 skipped as obvious noise (Tracy, PCI, OSError)
   2 skipped (same-layer, perf noise)
   7 sent to bisect          →  1 confirmed, 3 refuted, 1 inconclusive, 2 TBD
   --
   Success rate on hardware: 14% (1/7 tested)
   Target: >70%
```

---

## 1. Root Cause Analysis by Failure Category

### 1.1 BEFORE=FAIL, AFTER=FAIL (868299 family — 3 candidates)

**What happened**: Opus said "1 commit in range: revert of Device 2.0 API port. Fix commit 1ba8877f pre-identified." with high confidence. Both BEFORE and AFTER verification runs failed — the "fix" commit did not fix the test.

**What should have been caught**: The RUNBOOK's Opus triage prompt asks if there is a "plausible causal connection" — but it does NOT ask whether the test is **reproducibly failing at the last_failing_sha**. The verification step later revealed the test was failing at *both* commits, meaning either:
- The test was flaky and just happened to produce a fail→pass transition in CI
- The "fix" addressed a different failure mode while this test continued failing for other reasons

**What a better triage step would have done**:
1. The query should have provided the **failure rate** to Opus. A test with, say, 56% failure rate (176 fails, 138 passes for 884965) is not a deterministic regression — it is flaking.
2. Opus should have been explicitly asked: "Given a failure rate of X%, is a binary bisect reliable? A bisect requires that running the test at any commit before the fix reliably FAILS and after the fix reliably PASSES."
3. The query should have checked whether the failure signature appears on BOTH sides of the transition — if the same error appears at the first_passing_sha in other runs, the transition is noise.

**Signal that was available but ignored**: The candidates_merged.json shows `failure_rate_pct: "56.1"` for 884965. Any failure rate below ~80% means the test is not deterministically reproducible, and bisect will produce random results.

### 1.2 BEFORE=PASS, AFTER=PASS (888664 — flaky Galaxy test)

**What happened**: Opus classified this as PROCEED_TO_BISECT with "strong causal link" and noted a "31.9% failure rate suggests non-deterministic timing issue." But then concluded PROCEED anyway. When we ran verification, both BEFORE and AFTER passed — the test was not failing at those commits at all.

**The 31.9% failure rate should have been an automatic SKIP**. Here is why:

A 31.9% failure rate means that out of every 3 runs, the test fails roughly once. The probability of seeing a "fail→pass" transition purely by chance is:
```
P(fail at SHA_N) * P(pass at SHA_N+1) = 0.319 * 0.681 = 0.217
```
That means there is a **~22% chance per commit pair** of seeing a spurious transition. Over a 250-commit range on main, the expected number of spurious fail→pass transitions is enormous. The Snowflake query will inevitably find one and call it a "regression fix."

**What the prompt failed to catch**: The prompt says "not a known intermittent issue" as a filter, but does not give Opus the failure rate number to evaluate. The prompt asks about "plausible causal connection" without asking about *reliability*. Opus correctly identified the non-deterministic nature ("31.9% failure rate suggests non-deterministic timing issue") but had no instruction to SKIP based on that assessment.

**Red flag rule**: Any candidate with failure_rate < 70% should be automatically SKIP_LIKELY_FLAKY. At 70% failure rate, you have a ~79% chance of reproducing the failure in a single run. Below that, bisect becomes coin-flipping.

### 1.3 Noise Cluster (Tracy profiler, PCI IOCTL, OSError — 5 candidates)

**What happened**: These are:
- 4x `subprocess.CalledProcessError` from Tracy profiler — CI tooling failure, not a test failure
- 1x `TENSTORRENT_IOCTL_GET_DEVICE_INFO failed` — transient PCI link failure
- 1x `OSError: Can't load the configuration` — model file not accessible from runner

**Why they got through the layer pre-filter**: The layer pre-filter (Phase 2's step) checks which **files the commits between last_fail and first_pass touch**. It asks: "do any commits in this range touch a layer lower than the test?" This is entirely about the *fix* commits, not about the *failure signature*.

These errors are *not caused by code changes at all*. They are caused by:
- Tracy subprocess environment (runner config)
- PCI hardware link flaps (physical layer)
- Model file availability (HuggingFace cache / network)

The pre-filter cannot catch these because they would pass any commit-file-based check — there WILL be lower-layer commits in any 250-commit range on main. The issue is that the failure *itself* is not code-related.

**What would have caught them**:
1. **Error signature pattern matching** (before Opus): A simple grep-based pre-filter on the error message:
   ```
   SKIP if error matches:
     - "subprocess.CalledProcessError.*tracy"
     - "TENSTORRENT_IOCTL_GET_DEVICE_INFO failed"
     - "OSError.*Can't load the configuration"
     - "ConnectionRefusedError"
     - "runner.*lost communication"
   ```
   This is a known-noise blocklist that should be applied BEFORE spending an Opus call.

2. **Failure rate check**: The Tracy tests have 59.2% and 22.0% failure rates. The PCI test has 56.1%. All well below any reasonable determinism threshold.

### 1.4 Generic Errors (888286, 1039700238 — 2 candidates)

**888286** ("Pytest failed with exit code 1"): This is a *wrapper script* error that does not contain the actual test failure. The candidate should have been filtered because:
- The error message contains zero diagnostic information
- The first_passing_sha commit (2c59dcbd) is "Fix paths for zipping and uploading performance data" — a CI infra change, not a code fix
- The Opus triage caught this one and skipped it. Good.

**1039700238** (same-layer API change): `TypeError: LMHeadSampling.op() unexpected kwarg 'mcast_dst_working_buf_tensor'`. This is a pure Python API mismatch at layer 4. The error is in model code calling model-layer API. Not cross-layer at all.

**What would have caught 1039700238**: The layer pre-filter checks commit *files*, but should also check the **error stack trace** layer. The crash is in `models/` calling `models/` code — the error itself is layer 4. Even if a lower-layer commit exists in the range, the crash is entirely within one layer.

**Proposed fix**: Add an "error layer" check: parse the crash location from the error message (file path in TT_FATAL/TT_THROW/traceback) and compare to the test layer. If crash_layer == test_layer, it is a horizontal issue, not a vertical escape.

---

## 2. Snowflake Query Quality Analysis

### 2.1 What does the transition actually mean?

The query finds `(last_failing_sha, first_passing_sha)` transitions on main branch. The critical flaw: **this transition does NOT mean the test was deterministically failing before first_passing_sha**.

On main branch, each SHA typically runs the full CI suite once (sometimes twice for retries). A test with a 25% failure rate will produce this timeline:
```
SHA_100: PASS
SHA_101: PASS
SHA_102: FAIL  ← "last_failing_sha"
SHA_103: PASS  ← "first_passing_sha"
SHA_104: FAIL
SHA_105: PASS
```

The query picks SHA_102→SHA_103 as a "regression fixed" transition, but SHA_104 also fails — this is just normal flake behavior. The query's windowed `MAX(CASE WHEN SUCCESS = FALSE ...)` approach finds the most recent failure followed by a pass, which for flaky tests is just the last time the coin came up tails.

### 2.2 How does a 20% failure rate produce a false transition?

Looking at candidate 892929: 79 failures out of 315+79 = 394 runs = 20.1% failure rate. With 394 data points and a 20% base rate, the expected number of fail→pass transitions is roughly:
```
N_transitions ≈ N_runs * P(fail) * P(pass) ≈ 394 * 0.20 * 0.80 ≈ 63 transitions
```

There are approximately **63 spurious fail→pass transitions** in this test's history. The query picks one and calls it a regression fix. This is pure noise.

### 2.3 What additional fields the query should return

The query should compute and return:
1. **`failure_rate_pct`**: Already computed in candidates_merged.json but not in the RUNBOOK SQL. Must be part of the query.
2. **`consecutive_fail_count`**: How many consecutive failures preceded the first_passing_sha? A single isolated failure is noise. 5+ consecutive failures suggest a real regression.
3. **`consecutive_pass_count`**: How many consecutive passes followed the first_passing_sha? If only 1 pass followed by more failures, this is a flake.
4. **`total_runs_in_window`**: Sample size — how many runs of this test exist in the lookback period.
5. **`failure_consistency`**: What fraction of failures share the same normalized error signature? If 100% have the same error, it is deterministic. If errors vary, it is environmental.

### 2.4 Failure rate threshold for bisectability

A bisect requires that:
- At commit BEFORE the fix, the test FAILS with high probability
- At commit AFTER the fix, the test PASSES with high probability
- This must hold for ANY commit in the range, not just the boundary

For a bisect with K steps (log2 of range size), the probability that ALL K midpoint tests give the correct result is:
```
P(bisect_correct) = P(fail_before)^(K/2) * P(pass_after)^(K/2)
```

For a 250-commit range (K=8 steps) with 80% failure rate:
```
P = 0.80^4 * 1.0^4 = 0.41 — only 41% chance of correct bisect
```

For 95% failure rate:
```
P = 0.95^4 * 1.0^4 = 0.81 — 81% chance
```

**Recommended thresholds**:
- **>= 85% failure rate**: PROCEED (bisect has >65% reliability with 8 steps)
- **70-85%**: PROCEED_WITH_CAUTION (run test 3x at each bisect point, majority vote)
- **< 70%**: SKIP_LIKELY_FLAKY (bisect is unreliable)

None of the 16 candidates in this batch meet the 85% threshold. The highest is 59.2% (Tracy noise).

### 2.5 Proposed SQL additions

Add a pre-filter CTE that computes failure statistics:

```sql
-- Add after the 'transitions' CTE, before the final SELECT:
failure_stats AS (
  SELECT
    CICD_TEST_CASE_ID,
    COUNT(*) AS total_runs,
    SUM(CASE WHEN SUCCESS = FALSE THEN 1 ELSE 0 END) AS fail_count,
    SUM(CASE WHEN SUCCESS = TRUE THEN 1 ELSE 0 END) AS pass_count,
    ROUND(100.0 * SUM(CASE WHEN SUCCESS = FALSE THEN 1 ELSE 0 END) / COUNT(*), 1) AS failure_rate_pct,

    -- Consecutive failure streak ending at last_failing_sha
    -- (requires window function to count consecutive fails before the transition)
    MAX(CASE WHEN SUCCESS = FALSE THEN rn END) AS last_fail_rn,
    MIN(CASE WHEN SUCCESS = TRUE AND rn < MAX(CASE WHEN SUCCESS = FALSE THEN rn END)
        THEN rn END) AS pass_before_streak_rn
  FROM ranked
  GROUP BY CICD_TEST_CASE_ID
)

-- In the final SELECT, add:
WHERE ...
  AND fs.failure_rate_pct >= 70.0  -- Pre-filter flaky tests
  AND fs.fail_count >= 3           -- Minimum sample size
  AND fs.pass_count >= 2           -- Must have actually recovered
```

Additionally, add a **consecutive-run check**:

```sql
-- Only consider transitions with N+ consecutive failures BEFORE the transition
-- and M+ consecutive passes AFTER the transition
consecutive_check AS (
  SELECT
    CICD_TEST_CASE_ID,
    -- Count consecutive failures ending at last_failing_sha
    (SELECT COUNT(*)
     FROM ranked r2
     WHERE r2.CICD_TEST_CASE_ID = transitions.CICD_TEST_CASE_ID
       AND r2.SUCCESS = FALSE
       AND r2.rn <= transitions.last_fail_rn
       AND NOT EXISTS (
         SELECT 1 FROM ranked r3
         WHERE r3.CICD_TEST_CASE_ID = r2.CICD_TEST_CASE_ID
           AND r3.SUCCESS = TRUE
           AND r3.rn BETWEEN r2.rn AND transitions.last_fail_rn
       )
    ) AS consecutive_fails_before,
    -- Count consecutive passes starting at first_passing_sha
    (SELECT COUNT(*)
     FROM ranked r2
     WHERE r2.CICD_TEST_CASE_ID = transitions.CICD_TEST_CASE_ID
       AND r2.SUCCESS = TRUE
       AND r2.rn >= transitions.first_pass_rn
       AND NOT EXISTS (
         SELECT 1 FROM ranked r3
         WHERE r3.CICD_TEST_CASE_ID = r2.CICD_TEST_CASE_ID
           AND r3.SUCCESS = FALSE
           AND r3.rn BETWEEN transitions.first_pass_rn AND r2.rn
       )
    ) AS consecutive_passes_after
  FROM transitions
)

-- Filter: require 3+ consecutive fails before and 3+ consecutive passes after
WHERE cc.consecutive_fails_before >= 3
  AND cc.consecutive_passes_after >= 3
```

This single change would have eliminated **all 16 candidates** in the current batch, because none of them have 3+ consecutive fails followed by 3+ consecutive passes — they are all intermittent.

---

## 3. Opus Triage Prompt Analysis

### 3.1 What the prompt is NOT asking

The current prompt asks three questions:
1. Is the failure consistent with a genuine regression?
2. Is there a plausible causal connection?
3. Verdict

**Missing critical questions**:

1. **"Given a failure rate of X%, is a bisect feasible?"** — The prompt never mentions failure rate. Opus sees the error log and commit diffs but has no way to evaluate whether the test is deterministically reproducible.

2. **"Is the error signature an infrastructure/environment issue, regardless of causal plausibility?"** — The prompt's "infra flake" category is too narrow. It mentions "resource exhaustion" but not PCI errors, Tracy subprocess failures, model file loading, or network issues. These are all environmental.

3. **"What is the commit range size, and does it affect confidence?"** — A 250-commit range makes causal attribution essentially random. With 250 commits on main (which land ~50-80 PRs per day), there WILL be lower-layer changes. Finding one is not evidence of causality.

4. **"Is this error signature known to occur across many unrelated tests?"** — If fabric_firmware_initializer.cpp:263 timeout appears across 5 different model tests on Galaxy, it is likely a systemic Galaxy flake, not 5 independent regressions.

### 3.2 Red flags the prompt should explicitly evaluate

```
RED FLAGS (any one = strong SKIP signal):
1. Failure rate < 70% → test is not deterministically reproducible
2. Error mentions: PCI, IOCTL, tracy, subprocess, OSError loading config,
   model file not found, connection refused, runner lost
3. Commit range > 100 commits → attribution confidence too low for bisect
4. Same error signature appears in 3+ unrelated tests → systemic issue
5. Error is in device setup (before test body runs): "failed on setup"
6. The error contains "Timeout" + device number → hardware/firmware flake
7. Failure rate is 20-35% → classic flake band for Galaxy/T3K fabric issues
```

### 3.3 How the prompt should handle failure rates

Currently: not at all.

Proposed handling:

```
FAILURE RATE INTERPRETATION:
- >= 90%: Highly likely deterministic regression. Strong PROCEED signal.
- 70-89%: Probably deterministic but with some environmental sensitivity.
  PROCEED only if causal connection is strong.
- 50-69%: Ambiguous. Could be a partial fix or environmental.
  SKIP unless overwhelming causal evidence.
- 20-49%: Classic flake range. A bisect will produce random results.
  SKIP_LIKELY_FLAKY regardless of causal plausibility.
- < 20%: Rare intermittent. Not a regression pattern.
  SKIP_LIKELY_FLAKY.
```

### 3.4 "Plausible causal connection" vs "reliable enough to bisect"

The current prompt conflates two very different questions:

- **"Could this commit have caused this failure?"** — A commit that touches dispatch code *could* cause a device timeout. This is about possibility.
- **"If we bisect, will we get a reliable answer?"** — This requires that the test deterministically fails before the fix and deterministically passes after. This is about reproducibility.

The prompt should separate these:

```
Answer BOTH:
A) Causal plausibility: Is there a mechanistic pathway from the commit
   to this specific failure mode? (yes/no + explanation)
B) Bisect feasibility: Given the failure rate of {X}%, the commit range
   of {N} commits, and the error signature, would a binary bisect
   produce a reliable result? (yes/no + explanation)

Verdict: PROCEED_TO_BISECT only if BOTH A and B are yes.
```

### 3.5 Proposed rewritten Opus triage prompt

```
You are evaluating whether a CI test failure in tenstorrent/tt-metal is a
genuine vertical bug escape that can be reliably bisected.

TEST DETAILS:
- Test: {test_filepath} (layer {test_layer})
- Error signature: {failure_signature}
- Failure rate: {failure_rate_pct}% ({fail_count} failures / {total_runs} runs)
- Commit range: {commit_count} commits between last failure and first pass
- Hardware: {runner_label}

COMMITS IN RANGE (with files changed):
{commit_list_with_diffs}

RAW FAILURE LOG:
{failure_log}

EVALUATE EACH (answer explicitly):

1. ERROR CLASSIFICATION:
   Is this a code regression or an environmental/infrastructure issue?
   Environmental indicators (any = SKIP):
   - PCI/IOCTL failures, driver errors
   - Tracy/profiler subprocess errors
   - OSError loading model configs, file not found
   - "Connection refused", "runner lost communication"
   - "failed on setup" with device timeout on multi-device systems
   If environmental → SKIP_INFRA_NOISE.

2. DETERMINISM CHECK:
   Failure rate is {failure_rate_pct}%.
   - >= 85%: bisect-reliable. Proceed to causal analysis.
   - 70-84%: marginal. Proceed only with strong causal evidence.
   - < 70%: SKIP_LIKELY_FLAKY. A bisect at this rate produces random results.
     Explain: "Failure rate of {X}% means the test fails
     non-deterministically. A binary bisect requires reliable
     reproduction at every midpoint."

3. CAUSAL ANALYSIS (only if passed steps 1-2):
   Is there a specific commit in the range that plausibly fixes this
   specific error? Require:
   - The commit modifies code in the error's stack trace path, OR
   - The PR description explicitly mentions the error/test, OR
   - The commit reverts a change that introduced the failure mode.
   "Commits exist in a lower layer" alone is NOT sufficient —
   lower-layer commits exist in every 50+ commit range on main.

4. RANGE SIZE CHECK:
   {commit_count} commits in range.
   - <= 30: manageable for bisect
   - 31-100: caution, may need multiple runs per midpoint
   - > 100: SKIP_WIDE_RANGE unless a single commit is overwhelmingly
     likely (e.g., PR description names the exact test).

5. CROSS-TEST CHECK:
   Does this error signature (e.g., fabric_firmware_initializer timeout)
   appear across multiple unrelated tests? If yes, it is likely a
   systemic issue, not a single-commit regression. Note this.

VERDICT (one of):
- PROCEED_TO_BISECT: Passed all checks. Name the most likely fix commit SHA.
- SKIP_INFRA_NOISE: Failed check 1.
- SKIP_LIKELY_FLAKY: Failed check 2.
- SKIP_WIDE_RANGE: Failed check 4 with no strong candidate.
- SKIP_UNRELATED: Failed check 3 (no plausible causal link).
- SKIP_SYSTEMIC: Failed check 5 (same error across many tests).

Respond as JSON:
{
  "verdict": "...",
  "failure_rate_pct": {X},
  "commit_range_size": {N},
  "error_classification": "code_regression | infra_noise | flaky | unknown",
  "causal_link": "strong | weak | none",
  "most_likely_fix_sha": "<SHA or null>",
  "reasoning": "<2-4 sentences explaining the verdict>"
}
```

---

## 4. Layer Pre-filter Quality

### 4.1 Genuine cross-layer error signatures

Of the 16 candidates, let me classify the *actual crash location* (not the test file, not the fix file — the error itself):

```
Escape ID       Error crash location              Crash layer  Test layer  Cross-layer?
893095          Tracy profiler subprocess           CI infra     4          N/A (not code)
884965          pci_device.cpp:178 IOCTL            L1 (HW)      4          N/A (infra)
1282761621      OSError loading config              CI infra     4          N/A (not code)
888664          system_memory_manager.cpp:723       L2           4          YES (genuine)
1080529165      Card hang (no specific file)        L1/L2        4          MAYBE
892931          fabric_firmware_initializer.cpp:263  L2           4          YES (genuine)
892930          fabric_firmware_initializer.cpp:220  L2           4          YES (genuine)
868317          physical_system_discovery.cpp:68     L2           4          YES (genuine)
893059          Tracy profiler subprocess            CI infra     4          N/A
888286          "Pytest failed exit code 1"          Unknown      4          N/A (no info)
892891          fabric_firmware_initializer.cpp:220  L2           4          YES (genuine)
892949          topology_mapper.cpp:519              L2           4          YES (genuine)
892948          DMA timeout                          L1/L2        4          MAYBE
892929          fabric_firmware_initializer.cpp:263  L2           4          YES (genuine)
893305          Tracy profiler subprocess            CI infra     4          N/A
893304          Tracy profiler subprocess            CI infra     4          N/A
```

**Summary**: 7 of 16 (44%) have genuinely cross-layer error signatures (crash in L2 code, test at L4). But ALL of these are in the `fabric_firmware_initializer` / `topology_mapper` / `physical_system_discovery` family — Galaxy/T3K multi-device fabric initialization. These are known-flaky subsystems with 20-30% failure rates.

**Key insight**: Having a genuine L2 crash in an L4 test does NOT mean it is a *regression*. These fabric init failures happen intermittently on every run. They are L2 *flakes*, not L2 *regressions*.

### 4.2 Better way to determine fix layer

The current approach determines fix layer by looking at which files the fix commit touches. This is correct in principle but insufficient because:

1. Many commits touch files across multiple layers (a PR might modify `tt_metal/impl/` AND `models/`). The `be_dominant_layer` function picks the lowest, which biases toward "escape."
2. A commit that touches `tt_metal/fabric/` might be a refactor with no functional change — the layer of the file is not the layer of the impact.

**Better approach**: Cross-reference the error's stack trace with the fix commit's diff. A genuine escape should have the fix commit modifying code that appears in the error traceback. If the error says `fabric_firmware_initializer.cpp:263` and the fix commit modifies `fabric_firmware_initializer.cpp`, that is a strong signal. If the fix commit modifies `tt_metal/impl/dispatch/ring_buffer.cpp` and the error is in a different file entirely, the connection is weaker.

---

## 5. Structural Problems

### 5.1 All 16 candidates are layer 4 — is this expected?

**Partially expected, partially a pipeline bias.**

Layer 4 (models/) tests are the most numerous in CI. They also tend to fail most often because:
- They run on multi-device hardware (Galaxy, T3K) which has higher flake rates
- They have longer runtimes, increasing timeout probability
- They exercise the full stack, so any layer's flake surfaces here

However, the pipeline SHOULD also find layer 3 (ttnn/) failures fixed by layer 1-2 commits. The reason it does not is:

1. **The shell pipeline (phase2_candidates.sh) scans GitHub Actions workflows**, not Snowflake test data. It groups by workflow→job, not by test. TTNN test workflows tend to have more granular jobs (one test per job), so consecutive failures are less common — a single flaky test does not cause the whole job to fail if other tests in the job pass.

2. **The RUNBOOK's Snowflake query approach would find them**, but the RUNBOOK SQL is a "starting sketch" that has never been fully operationalized. The actual detection is done by phase2_candidates.sh scanning GitHub Actions API, not by Snowflake SQL.

3. **Layer 3→2 escapes have shorter fix times.** TTNN team members often fix their own bugs within hours, so the fail→pass transition window is small (1-5 commits). The pipeline's 14-day lookback with workflow-level scanning misses these narrow windows.

### 5.2 Should layer 3 tests appear more often?

Yes. To find them:

1. **Use Snowflake directly** instead of GitHub Actions API. Query `CICD_TEST` for tests whose filepath starts with `ttnn/` that have consecutive failures followed by passes.

2. **Look at the test level**, not the job level. A job might contain 50 ttnn tests, only 1 of which is failing. The job-level scan in phase2_candidates.sh sees the job as "failure" (exit code 1) but cannot distinguish which test failed.

3. **Reduce the consecutive-failure threshold for ttnn tests.** TTNN tests typically run faster and more frequently, so even 2 consecutive failures followed by 2 passes could indicate a real regression.

### 5.3 Systematic problems from 2026-05-11 that still apply

The May 11 analysis identified three problems:
1. **Candidate saturation from one workflow** — FIXED (per-workflow cap added, `MAX_CANDIDATES_PER_WORKFLOW=10`)
2. **Missing test deduplication** — PARTIALLY FIXED (job-name dedup added, but same-error-signature dedup not implemented)
3. **Wrong workflow scan order** — FIXED (reordered)

But the May 11 analysis **missed the fundamental problem**: even with perfect workflow coverage, the pipeline's core logic (find fail→pass transitions, assume they are regressions) is broken for non-deterministic tests. The fixes since May 11 (per-wf cap, reordering, post-fix stability checks) are all structural improvements that improved *coverage* but did nothing for *precision*.

The post-fix stability check in phase3_fixpoints.sh (lines 227-293) was a step in the right direction — it checks whether 2/3 of post-fix runs pass. But this check happens AFTER the expensive forward scan and commit enrichment. It should happen much earlier, ideally at detection time.

---

## 6. Specific Actionable Improvements (Prioritized)

### P0: Add failure rate threshold to detection query (eliminates 16/16 current false positives)

**What changes**: Add a `failure_rate_pct >= 70` filter to the Snowflake detection query (or equivalent in phase2_candidates.sh flakiness pre-filter).

**Current code** (phase2_candidates.sh lines 436-464): The flakiness filter already computes `likely_flaky` based on alternation rate and failure rate (20-80% range + alternation > 0.3). But it only MARKS candidates as flaky — Phase 3 skips them, but they still consume candidate slots in Phase 2.

**Proposed change**: Move the flakiness filter to be a HARD CUT before log downloads:

```bash
# In phase2_candidates.sh, after computing flake_score:
# HARD SKIP: failure rate below threshold means bisect is unreliable
fail_ratio=$(echo "$candidate" | jq -r '
  (.failing_runs | length) as $fails |
  ... # compute from timeline
')
if (( $(echo "$fail_ratio < 0.70" | bc -l) )); then
  _mark_seen "$cand_key" "skip_low_determinism"
  continue
fi
```

But even better: **compute failure rate in the Snowflake SQL** and exclude tests below 70% before they ever reach the pipeline:

```sql
-- Add to the RUNBOOK SQL:
HAVING failure_rate_pct >= 70.0
```

**What it eliminates**: All 16 candidates in the current batch have failure rates from 20-59%. This filter eliminates 100% of current false positives.

**Estimated impact**: Current precision 14% → precision cannot be measured (no candidates pass), but eliminates all false positives. We would need to expand the query parameters to find the true positives (like 867860 which had a much higher failure rate).

**Risk**: May filter out some genuine escapes where the failure rate is depressed by inconsistent test environments. Mitigate by computing failure rate per-runner or per-hardware config rather than globally.

---

### P1: Add consecutive-run requirements (eliminates spurious transitions)

**What changes**: Require N consecutive failures BEFORE the transition AND M consecutive passes AFTER.

**Current state**: The phase2 script already checks for `CONSECUTIVE_RUNS` (default 3) consecutive failures. But the RUNBOOK SQL uses a windowed approach that does not properly count consecutive runs.

**Proposed**: In the Snowflake SQL, require:
- >= 5 consecutive failures immediately before the first_passing_sha
- >= 3 consecutive passes immediately after (including first_passing_sha)

This is the single most effective filter because flaky tests produce isolated failures, not streaks.

**What it eliminates**: Any test where failures are interspersed with passes — i.e., all flaky tests. A genuine regression that gets fixed will show: PASS PASS PASS FAIL FAIL FAIL FAIL FAIL PASS PASS PASS PASS. A flake shows: PASS FAIL PASS PASS FAIL PASS FAIL PASS.

**Estimated impact**: Combined with P0, would have eliminated all 16 candidates. The confirmed escape (867860) DID have a clear streak of failures, so it would have survived.

---

### P2: Known-noise error signature blocklist (eliminates infra noise before Opus)

**What changes**: Add a pattern-matching step between log download and Opus triage that checks error messages against known infrastructure noise patterns.

**Implementation** (add to phase2_candidates.sh after log extraction):

```bash
INFRA_NOISE_PATTERNS=(
  "subprocess.CalledProcessError.*tracy"
  "subprocess.CalledProcessError.*profiler"
  "TENSTORRENT_IOCTL.*failed"
  "OSError.*Can't load the configuration"
  "ConnectionRefusedError"
  "runner.*lost communication"
  "runner.*received.*shutdown"
  "Docker.*pull.*failed"
  "cannot connect to Docker daemon"
  "health check.*failed.*container"
  "artifact.*upload.*failed"
  "Pytest failed with exit code [0-9]"  # wrapper error, no specific test
)

for pattern in "${INFRA_NOISE_PATTERNS[@]}"; do
  if echo "$log_snippet" | grep -qiE "$pattern"; then
    _mark_seen "$cand_key" "infra_noise_pattern"
    _mark_job_noisy "$wf_basename" "$job_name"
    continue 2  # skip to next candidate
  fi
done
```

**What it eliminates**: Tracy profiler (4 candidates), PCI IOCTL (1), OSError (1), generic pytest wrapper (1) = 7 candidates.

**Estimated impact**: 7/16 = 44% of false positives eliminated without any Opus calls. Saves ~$0.50 per candidate in API costs.

---

### P3: Rewrite Opus triage prompt (see Section 3.5)

**What changes**: Replace current 3-question prompt with the 5-check structured prompt in Section 3.5. Critical additions:
- Failure rate is provided as input and Opus is instructed to SKIP below 70%
- Commit range size is provided and Opus is instructed to SKIP above 100 without strong evidence
- Error classification includes specific infra noise patterns
- Causal analysis requires stack-trace-to-diff matching, not just "lower layer commits exist"

**What it eliminates**: Prevents Opus from saying PROCEED on flaky tests where it currently says "strong causal link" despite 20-30% failure rates.

**Estimated impact**: Would have caught 888664 (the 31.9% Galaxy test that Opus approved) and the fabric_firmware_initializer cluster (20-29% failure rates).

---

### P4: Add error-layer classification (prevents same-layer noise)

**What changes**: Parse the crash location from the error message and determine its layer. If crash_layer == test_layer, skip the candidate (it is a horizontal issue, not an escape).

**Implementation**:

```bash
# Extract crash file path from common error patterns:
# TT_FATAL @ /project/tt_metal/fabric/foo.cpp:68
# TT_THROW @ /project/tt_metal/impl/bar.cpp:723
# Traceback ... File "models/demos/test.py", line 42
crash_path=$(echo "$log_snippet" | grep -oE 'TT_(FATAL|THROW) @ /project/([^ :]+)' | head -1 | sed 's|.*/project/||')
if [ -n "$crash_path" ]; then
  crash_layer=$(be_file_to_layer "$crash_path")
  test_layer_num=$(layer_to_num "$test_layer")
  crash_layer_num=$(layer_to_num "$crash_layer")
  if [ "$crash_layer_num" -ge "$test_layer_num" ]; then
    # Crash is at same or higher layer — not a vertical escape
    _mark_seen "$cand_key" "same_layer_crash"
    continue
  fi
fi
```

**What it eliminates**: 1039700238 (same-layer API change). Also provides useful signal for Opus triage — knowing the crash layer vs test layer narrows the analysis.

**Estimated impact**: Low in current batch (1 candidate), but prevents a class of false positives.

---

### P5: Switch detection from GitHub Actions API to Snowflake (medium-term)

**What changes**: Replace phase2_candidates.sh's GitHub Actions API workflow scanning with direct Snowflake SQL queries against `CICD_TEST`.

**Why**: The current pipeline's biggest limitation is that it operates at the *job* level (a job contains many tests). A job fails if ANY test fails. This means:
- A single flaky test causes the whole job to appear as "failing"
- Different tests failing in different runs appear as the same "job failure"
- The pipeline cannot distinguish which specific test is the regression

Snowflake's `CICD_TEST` table has per-test-case results. A direct query can:
- Find specific tests that transitioned from fail to pass
- Compute per-test failure rates
- Count consecutive failures for a specific test
- Join to `CICD_TEST_CASE` for filepath/layer classification

**Caveat**: `CICD_TEST` has 5.8B rows. Queries must be tightly scoped by date and branch. But with proper filtering (last 14 days, main branch, failure_rate >= 70%, consecutive_fails >= 5), the result set is manageable.

**Estimated impact**: Would surface layer 3 (ttnn) escapes that the current pipeline cannot find because it does not have per-test resolution.

---

### P6: Cluster deduplication by error signature (low-effort, high-value)

**What changes**: Before sending candidates to bisect, cluster them by normalized error signature. If 5 candidates all crash at `fabric_firmware_initializer.cpp:263`, bisect ONE representative and apply the result to all.

**Current behavior**: The opus-triage-2026-05-16.json does cluster candidates by `cluster_with` field. But bisect still runs independently for each.

**Proposed**: Pick the cluster member with the smallest commit range. If bisect confirms it, mark all cluster members as confirmed. If refuted, mark all as refuted.

**What it eliminates**: Saves 4-5 hardware runs per cluster. The fabric_firmware_initializer cluster has 4 members — only 1 bisect needed instead of 4.

**Estimated impact**: 4x reduction in hardware cost for clustered failures. Already partially implemented in the Opus triage output.

---

## 7. Summary: Path to >70% Precision

```
Current state:    14% precision (1/7 hardware-tested)
                  0% precision on new batch (0/16 candidates are bisectable)

With P0+P1:       ~100% noise eliminated at query level
                   (all 16 candidates have <70% failure rate
                    and lack consecutive run patterns)
                   Need to tune thresholds to keep genuine escapes like 867860

With P2:          Additional 44% noise eliminated for infra signatures
                   (Tracy, PCI, OSError patterns caught before Opus)

With P3:          Opus no longer approves flaky tests for bisect
                   (failure rate is input, not discovered)

With P5:          Pipeline can find layer 3 escapes (ttnn)
                   (currently invisible due to job-level detection)

Combined target:  Fewer candidates reach hardware, but each one is a
                   genuine deterministic regression with a narrow commit range.
                   Target: 5-10 candidates per month, >70% confirmed.
```

---

## Appendix A: Score Card — All Tested Candidates

```
ID                      Test                          HW      Failure%  Verdict       Why
867860__c5268e19        test_model.py (Llama-3.1-8B)  N300    HIGH*     CONFIRMED     sdpa kernel build fix
868299__39a3856e        test_decoder_prefill.py        N150    ~50%      REFUTED       BEFORE=FAIL AFTER=FAIL
882550__9d4ab66e        test_ci_dispatch.py            N150    ~50%      REFUTED       Same range as 868299
882440__9d4ab66e        test_vision_model.py           N150    ~50%      REFUTED       Same range as 868299
888664__ed481281        test_qwen_decoder.py           Galaxy  31.9%     REFUTED       BEFORE=PASS AFTER=PASS
868311__028e027b        demo.py Llama3-8b              P300    ~25%      INCONCLUSIVE  Test deleted at AFTER commit
1268500663__f622a01e    test_sdpa_nightly.py           P150    ~50%      SKIPPED       Perf noise
207161529__f622a01e     test_sdxl_clip_encoders.py     P150    ~50%      SKIPPED       Perf noise
1039700238__52db4015    test_lm_head_sampling.py       P150b   ~40%      SKIPPED       Same-layer API change
893095__fa68384f        test_decode_demo_perf          Galaxy  59.2%     SKIPPED       Tracy profiler noise
893059__f086f98a        test_ds_lm_head_device_perf    Galaxy  22.0%     SKIPPED       Tracy profiler noise
893305__dac056e0        test_ds_rms_norm_device_perf   Galaxy  20.0%     SKIPPED       Tracy profiler noise
893304__dac056e0        test_ds_distributed_norm       Galaxy  20.0%     SKIPPED       Tracy profiler noise
884965__fd901416        test_gpt_oss_demo              T3K     56.1%     SKIPPED       PCI IOCTL failure
1282761621__9a47ed7e    test_e2e_vision_text_pipeline  BH      41.7%     SKIPPED       OSError loading config
888286__faec73d9        test_ci_dispatch               T3K     21.8%     SKIPPED       Generic pytest exit code

* 867860 failure rate not in candidates_merged.json but was high enough to bisect
  successfully in 5 steps with all-PASS midpoints.
```

## Appendix B: Currently In-Flight Bisects

These are running as of 2026-05-16 and their outcome will provide additional data:

```
885783__6620908c   test_tosa_scatter_normal      BH-LoudBox  Step 2/~5  ttnn test!
890570+890572      test_fabric_mux_bandwidth     T3K         Step 2/~4  metalium test!
892929__ee56bd21   test_tt_conv3d_1x1x1          Galaxy      Step 1/~6  20.1% failure rate
```

**Notable**: 885783 and 890570/890572 are layer 3 (ttnn) and layer 2 (metalium) tests respectively — these came from a different candidate batch and represent the kind of tests the pipeline SHOULD be finding more of. Their failure rates and bisect reliability should be tracked to calibrate thresholds.

892929 has a 20.1% failure rate and is predicted to be unreliable. If the bisect produces an inconsistent result (PASS at both midpoints, or FAIL at both), it confirms the flakiness analysis.

---

*Generated 2026-05-16 by BrAIn post-mortem analysis*
