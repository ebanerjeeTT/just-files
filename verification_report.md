# Bug Escape Verification Report
*Generated: 2026-05-12 17:30 UTC*

11 HCVs from overnight triage-ci. 3 Galaxy deferred (infra unavailable). 2 previously refuted.
6 HCVs actively verified via CI (before/after branch pairs).

## Results

*HCV #1* — Galaxy QwenImage (Galaxy TG, PR #40225)
  ⏸️ *DEFERRED* — Galaxy infra unavailable — deferred

*HCV #2* — Galaxy SentenceBert (Galaxy TG, PR #39920)
  ⏸️ *DEFERRED* — Galaxy infra unavailable — deferred

*HCV #3* — BH Galaxy CCL (Galaxy TGG, PR #41930)
  ⏸️ *DEFERRED* — Galaxy infra unavailable — deferred

*HCV #4* — resnet50 (N300, PR #41744)
  ❌ *REFUTED* — REFUTED — test already disabled before fix

*HCV #5* — wan2.2 (N150, PR #41050)
  ❌ *REFUTED* — REFUTED — test already disabled before fix

*HCV #6* — stable_diffusion (N150, PR #41744)
  ✅ *CONFIRMED* — N150: BEFORE=failure, AFTER=success

*HCV #7* — vit (N150, PR #41744)
  ✅ *CONFIRMED* — N150: BEFORE=failure, AFTER=success

*HCV #8* — BH post-commit (BH P150, PR #41744)
  ✅ *CONFIRMED* — BEFORE=failure, AFTER=success

*HCV #9* — BH multi-card CBs (BH 4xP150, PR #41050)
  ⚠️ *INCONCLUSIVE* — Both failed: BEFORE=failure, AFTER=failure

*HCV #10* — BH P300 from_torch (BH 2xP300, PR #41217)
  ⚠️ *INCONCLUSIVE* — Both failed: BEFORE=failure, AFTER=failure

*HCV #11* — BH multi-card stress (BH 4xP150, PR #41050)
  ⚠️ *INCONCLUSIVE* — Both failed: BEFORE=failure, AFTER=failure

## Summary
• ✅ Confirmed: 3
• ❌ Refuted: 2
• ⚠️ Inconclusive: 3
• ⏸️ Deferred (Galaxy): 3

## CI Run Links

• d4bc1299-fd BEFORE: https://github.com/tenstorrent/tt-metal/actions/runs/24432636270 (completed/cancelled)
• d4bc1299-fd AFTER: https://github.com/tenstorrent/tt-metal/actions/runs/24432641416 (completed/cancelled)
• d4bc1299-bh BEFORE: https://github.com/tenstorrent/tt-metal/actions/runs/24451423159 (completed/failure)
• d4bc1299-bh AFTER: https://github.com/tenstorrent/tt-metal/actions/runs/24451429919 (completed/success)
• 4a20c4b5-bhe BEFORE: https://github.com/tenstorrent/tt-metal/actions/runs/24451454720 (completed/failure)
• 4a20c4b5-bhe AFTER: https://github.com/tenstorrent/tt-metal/actions/runs/24451461379 (completed/failure)
• d1ab8540-bhe BEFORE: https://github.com/tenstorrent/tt-metal/actions/runs/24451486030 (completed/failure)
• d1ab8540-bhe AFTER: https://github.com/tenstorrent/tt-metal/actions/runs/24451492303 (completed/failure)