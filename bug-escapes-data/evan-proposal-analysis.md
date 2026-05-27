# Analysis: Evan's Bug Escape Detection Redesign Proposal

**Date**: 2026-05-14
**Context**: After a campaign that produced 0 confirmed escapes from 15 candidates (7 refuted, 7 inconclusive, 1 unverifiable)

---

## 1. Signal Quality: The 5+5 Consecutive Threshold

**What it solves (well):**
- The current campaign's 7 refutations were ALL caused by BEFORE=PASS — the algorithm misidentified the fix commit because it was working from thin data (1-2 failure samples before the transition). A 5+5 rule would have filtered out every one of those 7 false positives, since none of them had 5 consecutive identical failures.
- Flaky tests (which generated 4 of our Phase 3 skips) inherently cannot produce 5 consecutive identical failures. The 5+5 rule subsumes the current flaky-test filter.
- Hardware timeout signatures like `TT_METAL_OPERATION_TIMEOUT_SECONDS` — which dominated this campaign's candidates — are intermittent. They would rarely hit 5 consecutive. This is a feature, not a bug: timeout hangs are usually not caused by a single commit and are poor escape candidates.

**What it costs:**
- Real escapes can be fixed fast. In the previous successful campaign (4 confirmed escapes), some had only 2-3 failure samples before the fix landed. A strict 5+5 rule would have missed those.
- Nightly-only workflows (blackhole-e2e-tests, tt-metal-l2-nightly) run ~1x/day. 5 consecutive failures = 5 calendar days minimum. If a bug is fixed within 3 days (common for P0s), we never see 5 failures.
- Workflows with sparse schedules (weekly perf sweeps, monthly release tests) become essentially invisible.

**Recommendation:**
- 5+5 is too aggressive. A 3+3 threshold would have still filtered all 7 refuted candidates from this campaign (none had 3 consecutive failures) while preserving detectability for bugs fixed within a week.
- Better yet: make it configurable per-workflow based on run frequency. Daily workflows → 3+3, post-commit (runs per push) → 5+5, nightly → 2+2.

---

## 2. LLM Attribution Accuracy

**When it works well:**
- Clean single-commit PRs with descriptive titles and obvious file paths. Example from our ledger: PR #43648 "Fix UB float<->uint32_t bitcast in LLK SFPU headers" — an LLM would correctly identify this as the fix for an LLK-layer test failure involving data format issues.
- Cases where the commit message explicitly references the failing test or error signature.

**When it fails:**
- **Merge commits with many files**: A merge-queue squash touching 50 files across layers is ambiguous. The LLM would have to guess which of several changes fixed the test.
- **Indirect fixes**: PR #43103 "Fix dispatch_s noc1 barrier issue" — this is a metalium dispatch fix that happens to unblock a fabric unit test. The connection is non-obvious even to domain experts. An LLM reading the diff would see NOC register manipulation and might not connect it to a test named "fabric and ttnn UDM unit tests."
- **Revert-then-reland patterns**: Common in tt-metal. A commit reverts a broken change, then a later commit relands with a fix. The LLM might attribute to the reland when the revert was the actual fix point.
- **Infrastructure changes**: CMake, CI YAML, runner config changes can fix test failures by changing build flags or test ordering. An LLM would likely dismiss these as non-fix commits.

**Risk assessment:**
- For the "is this a vertical escape?" classification specifically, the LLM only needs to identify WHICH LAYER the fix commit touches, not the full causal chain. This is much easier — file paths alone usually determine the layer (tt_llk/ → tt-llk, tt_metal/ → tt-metalium, ttnn/ → ttnn, models/ → models).
- The bigger risk is false NEGATIVES: the LLM incorrectly dismisses the actual fix commit and picks the wrong one, causing the pipeline to test the wrong BEFORE/AFTER boundary and waste a verification slot.

**Recommendation:**
- Use the LLM for layer classification (high accuracy from file paths) but keep the current bisection-style fix-commit identification (first commit where test transitions to passing) as the primary attribution method.
- LLM serves as a CONFIDENCE BOOSTER, not the sole attributor. If the LLM's pick matches the bisection result, confidence=high. If they disagree, confidence=low and flag for human review.

---

## 3. End-to-End Pipeline Analysis

**Proposed 4-phase flow:**
```
Phase 1: Detection (5+5 filter)
  → Phase 2: LLM Attribution (Opus reads diffs)
    → Phase 3: Escape Classification (layer check)
      → Phase 4: Verification (CI runs)
```

**Logical issues:**

*Phase ordering is suboptimal.* The layer check (Phase 3) is cheap and deterministic — it should run BEFORE the expensive LLM attribution (Phase 2). No point asking Opus to analyze 20 commits if the failing test and all commits in the range are in the same layer (no possible escape).

**Better ordering:**
```
Phase 1: Detection (consecutive failure/pass threshold)
  → Phase 2: Layer pre-filter (are fix candidates in a lower layer?)
    → Phase 3: LLM Attribution (only for cross-layer transitions)
      → Phase 4: Verification (CI runs)
```

This cuts LLM costs significantly. In our current campaign, 1 of 15 candidates had a same-layer fix (PR #43449 "[CI Fix] Update bcast core" — models-layer fix for a models-layer test). A layer pre-filter would have eliminated it before burning Opus tokens.

**Failure modes:**
- Phase 1 (5+5) produces zero candidates during quiet CI periods → entire pipeline stalls. Need a fallback: if no 5+5 candidates exist after N days, relax to 3+3.
- Phase 2 (LLM) is the latency bottleneck. Each Opus call with full diff context takes 30-60 seconds and costs ~$0.15-0.50. With 10 candidates × 20 commits each = 200 LLM calls per detection run.
- Phase 3 (layer check) can misclassify files that live at layer boundaries. Example: `ttnn/cpp/ttnn/operations/transformer/sdpa/device/kernels/` — is this ttnn or tt-metalium? The current heuristic uses path prefixes, which works for 95% of cases.

---

## 4. Root Cause Comparison

**What went wrong with the current approach:**

The core failure mode was **thin signal + wrong fix-commit attribution**. Here's the causal chain:

```
1. Test fails 1-2 times (thin data)
2. Test passes (could be: fix landed, flaky recovery, infra change, different runner)
3. Algorithm picks "first passing commit" as fix commit
4. But the test was already passing BEFORE that commit (BEFORE=PASS → refuted)
```

The root cause is that 1-2 failure samples cannot distinguish "genuine regression fixed by commit X" from "transient failure that self-resolved."

**Does Evan's proposal address this?**

Yes, directly. The 5+5 rule attacks the root cause: requiring 5 consecutive identical failures ensures the pattern is a genuine regression (not flaky), and requiring 5 consecutive passes ensures the fix is stable (not a lucky run). The failure-to-pass transition point is much more likely to correspond to the actual fix commit when both sides have depth.

However, the proposal does NOT address the second root cause: **hardware availability**. 7 of 15 candidates were inconclusive because P300-viommu and BH-LoudBox runners are only available during nightly windows. The 5+5 rule reduces the NUMBER of candidates reaching verification, which partially helps (fewer candidates competing for scarce hardware slots), but it doesn't solve the fundamental constraint.

---

## 5. Hardware Constraint Impact

**Current state:** 7 of 15 = 47% inconclusive due to hardware unavailability.

**How the new pipeline affects this:**

*Helps:*
- Fewer candidates reach Phase 4. If 5+5 + layer pre-filter + LLM attribution reduces candidates from 15 to 3-5 per campaign, and hardware can handle 3-5 verification runs in a nightly window, the inconclusive rate drops significantly.
- Higher-confidence candidates mean each verification slot is better utilized.

*Hurts:*
- The 5+5 rule requires nightly-frequency workflows to accumulate 5 failures over 5 days before a candidate even enters the pipeline. By the time it reaches Phase 4 (add LLM analysis time + dispatch queue), the fix could be 7-10 days old. At that point, the BEFORE commit is very stale, and the probability of unrelated breakage confounding the BEFORE run increases.

*Net assessment:* Modest improvement. The real fix for hardware availability is either (a) requesting dedicated runner capacity for verification, or (b) implementing a scheduling system that pre-books nightly windows for verification runs.

---

## 6. Edge Cases and Risks

**False positives that survive the new pipeline:**
- **Dependent failures**: Test A fails because test B (which runs first in the same job) corrupts shared state. Test B gets fixed → test A starts passing. If test B's fix is in a lower layer, this registers as a vertical escape, but it's actually a test-ordering dependency, not a genuine escape.
- **Build flag changes**: A CMake change enables/disables a feature flag, causing tests to flip. Not a bug escape, but the LLM might attribute it as a fix if it changes behavior.
- **Rollback-reland cycles**: Commit X breaks test, commit X+3 reverts X, commit X+10 relands with fix. The 5+5 window might span the revert, and the LLM sees the reland as the fix. Technically correct, but the "fix" is really "undo damage from the same layer" — not an escape.

**False negatives (real escapes we'd miss):**
- **Fast fixes**: Bug introduced and fixed within 2-3 commits (< 5 failures). This is the most concerning gap. Fast P0 fixes are exactly the kind of escapes we WANT to find — a lower-layer bug serious enough to get fixed immediately.
- **Intermittent hardware failures**: A metalium bug that only manifests on certain board revisions. Might produce 3 failures, 1 pass, 2 failures, then get fixed. The non-consecutive pattern breaks the 5+5 rule even though it's a real escape.
- **Multi-fix cascades**: Test needs fixes in two layers (e.g., LLK fix + TTNN workaround). Only 3 failures from the LLK bug, then 2 more from the TTNN issue. Neither window hits 5.

---

## 7. Alternative Suggestions

**A) Hybrid threshold with confidence scoring (recommended)**

Instead of a hard 5+5 cutoff, assign a confidence score:
```
Score = (consecutive_failures * 2) + (consecutive_passes * 2) + signature_match_bonus
  - signature_match_bonus = 5 if all failure signatures are identical
  - Threshold: score >= 14 → auto-verify
  - Score 8-13 → LLM triage (Opus decides if worth verifying)
  - Score < 8 → skip
```
This preserves the ability to catch 3-failure escapes while still filtering out 1-2 sample noise.

**B) Signature clustering before counting**

The "identical" in "5 consecutive identical failures" needs careful definition. Current failure signatures are noisy — they include timestamps, memory addresses, and runner-specific paths. Normalize signatures before comparison:
- Strip timestamps and absolute paths
- Collapse stack traces to top-3 frames
- Group by (test_name, normalized_signature_hash)

This prevents a real regression from being split into "different" failures due to cosmetic signature differences.

**C) LLM as pre-filter, not attributor**

Instead of having the LLM pick the fix commit, have it answer a simpler yes/no question per candidate:

> "Given this test failure pattern and these N commits in the transition window, is it plausible that one of these commits fixed this failure? Or is this more likely a flaky test / infra recovery?"

This binary question is much more reliable than attribution and serves as a cheap filter before burning verification hardware.

**D) Adaptive lookback based on workflow frequency**

```
Workflow runs per day    Min consecutive failures
  >= 4 (post-commit)       5
  1-3 (daily/nightly)      3
  < 1 (weekly/sparse)      2
```
This normalizes the detection sensitivity across workflow frequencies.

**E) Parallel BEFORE/AFTER with cancellation**

For the hardware constraint: dispatch BEFORE and AFTER runs simultaneously. If BEFORE=PASS (refuted), cancel the AFTER run immediately. This halves wall-clock time for refuted candidates and frees hardware faster.

---

## Summary

```
Aspect                    Current Approach    Evan's Proposal     Verdict
Signal filtering          Weak (1-2 samples)  Strong (5+5)        Good direction, but 5+5 too strict
Fix attribution           Bisection           LLM (Opus)          Risky as sole method; better as supplement
Phase ordering            Detect→Verify       4-phase             Good, but swap Phase 2 and 3
False positive rate       High (47% refuted)  Low (estimated)     Major improvement
False negative risk       Low                 Medium-High         Concerning for fast-fixed P0 bugs
Hardware utilization      Poor (47% wasted)   Better (fewer cands) Modest improvement
Cost per run              Low (no LLM)        High (Opus calls)   Need cost controls
```

**Bottom line:** Evan's proposal correctly diagnoses the root cause (thin signal → wrong fix attribution → wasted verification) and the 5+5 consecutive threshold is the right conceptual direction. But the specific threshold is too aggressive for nightly workflows, and LLM attribution should supplement rather than replace bisection-based fix identification.

The highest-impact changes in priority order:
1. Implement adaptive consecutive-failure thresholds (3-5 based on workflow frequency)
2. Move layer pre-filter BEFORE LLM attribution (cheap filter first)
3. Use LLM as a confidence booster / plausibility filter, not sole attributor
4. Normalize failure signatures before counting consecutiveness
5. Dispatch BEFORE/AFTER in parallel with early cancellation

---

*Analysis by BrAIn, 2026-05-14*
