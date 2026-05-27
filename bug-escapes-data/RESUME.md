# Bug Escape Campaign — Resume State
_Written 2026-05-16 at Evan's request. Resume when told._

---

## No cron jobs to restart
There were no active scheduled cron jobs at pause time (CronList returned empty).
If a bi-hourly poll was set up in a prior session it had already expired.

---

## Active GitHub Actions runs (DO NOT CANCEL)

| Run ID | Escape | Machine | Step | Commit | Status at pause |
|--------|--------|---------|------|--------|-----------------|
| 25964583087 | 885783 tosa_scatter | BH-LoudBox | mid2 @ 0d73a28d (4/9) | Fix SDPA SFPI rounding mode | in_progress |
| 25964568086 | 892929 Mochi conv3d | Galaxy | mid1 @ 6c76a394 (25/50) | Replace sfpi::setsgn with sfpi::copysgn | in_progress |
| 25964730548 | 890570+572 fabric_mux_bw | T3K | mid2 @ f17049d5 (3/7) | Fix sdpa decode core alloc | in_progress |
| 25964260250 | 867860 test_model N300 | N300 | VERIFY AFTER @ 5ed11e22 | PR #42426 fix | PASS (verified) |

Run 25964257929 (867860 BEFORE) completed = FAIL (expected, confirms bisect).
Run 25964260250 (867860 AFTER) completed = PASS → 867860 verification CONFIRMED.

---

## Next steps for each active run (process when I resume)

### 885783 (BH-LoudBox, mid2 @ 0d73a28d idx 4/9)
- If PASS → fix in lower half (idx 0-3). New mid = (0+3)//2 = 1 = `32d20f37`... wait wrong range.
  - Range is 21 commits (6620908c → 438eaf0d). At pause: low=0, high=9, mid2=idx4.
  - If PASS → high=3, mid3 = (0+3)//2 = 1 = commit at idx 1.
  - If FAIL → low=5, mid3 = (5+9)//2 = 7 = commit at idx 7.
  - Opus suspect: `e76ffe15` at idx 3 (Fix transpose common-runtime-args CRT refresh for interleaved buffers).
  - If mid2 PASS → e76ffe15 enters scope (idx 0-3). Very likely to be the fix.
- Fetch 885783 range from `/tmp/885783_range.json` if still present, or re-fetch:
  `GET /repos/tenstorrent/tt-metal/compare/6620908ce8aa079158511309fc418c68183d4d01...438eaf0d8161fe141f8f893434b3ca375ca1fa03`

### 890570+572 (T3K, mid2 @ f17049d5 idx 3/7)
- Range: 16 commits (7258725a → 2c1b1344). At pause: low=0, high=7, mid2=idx3.
- Commits 0-7 from `/tmp/890570_range.json`:
  - [0] 70277d3a Revert umd bumps (#42945) — SUSPICIOUS for fabric mux BW
  - [1] 32d20f37 Physically Connected Cores vs Logical Connected Cores
  - [2] 78f3f9d6 ttnn.slice fix
  - [3] f17049d5 Fix sdpa decode core alloc (mid2 being tested)
  - [4] 3ff12678 Restore Conv3D block config
  - [5] 034ac587 fused linear kernel broadcast fix
  - [6] e9c1b63b tile dims alignment
  - [7] 9c9c10d7 fix up cb usage in sampling
- If PASS → high=2, mid3 = (0+2)//2 = 1 = `32d20f37`
- If FAIL → low=4, mid3 = (4+7)//2 = 5 = `034ac587`
- Best guess: idx 0 `70277d3a` "Revert umd bumps" is the fix (UMD revert → fabric mux BW recovery)

### 892929 (Galaxy, mid1 @ 6c76a394 idx 25/50)
- Range: 51 commits (ee56bd21 → c63bb847). At pause: low=0, high=50, mid1=idx25.
- Commit at idx 25: `6c76a39463bb777e79350d79001a44a1dbdc275f` "Replace sfpi::setsgn with sfpi::copysgn"
- Notable commits in range (from `/tmp/892929_range.json`):
  - idx 24: `25155aac` "Update BH ethernet firmware API" — very suspicious for fabric_firmware_initializer.cpp crash
  - idx 31: `13e2a896` "Read updated pending interrupts on txn ids"
  - idx 50: `c63bb847` "UMD Bump 13.05.2026" (= first_passing)
- If PASS → high=24, next mid = 12. If idx 24 (25155aac) is fix, bisect will find it in 2-3 more steps.
- If FAIL → low=26, next mid ≈ 38.
- This escape clusters with 892930/892931/892891 — same root cause. Resolving 892929 likely resolves the whole cluster.
- Runner label for Galaxy: `["arch-wormhole_b0", "pipeline-functional", "topology-6u"]`

### 867860 (N300 verification — COMPLETE)
- BEFORE=FAIL, AFTER=PASS. Verification CONFIRMED. No action needed.
- Already recorded in campaign-state.json as verified.
- Fix: PR #42426 "ttnn: fix sdpa kernel build" commit 5ed11e22872f0048a4f3d10383810084ec2c91a5
- Should file this as a confirmed vertical escape report (optional next step when resuming).

---

## Pending candidates (7, queued in campaign-state.json)

Dispatch priority when hardware becomes available:

**T3K (when 890570+572 completes):**
1. `892931__f3fed7ba` — test_tt_decoder_forward, Mochi VAE, T3K, 250 commits, fabric_firmware_initializer.cpp:22
2. `868317__ee56bd21` — simple_vision_demo, Llama3.2-90B T3K, 52 commits, physical_system_discovery.cpp:68 FATAL

**Galaxy (when 892929 completes):**
3. `892930__f3fed7ba` — test_tt_upsample_forward, Mochi VAE, Galaxy, 250 commits
4. `892891__fb9d2525` — test_single_transformer_block, Flux1, Galaxy, 250 commits
5. `892949__fd218c88` — test_pipeline_sd35, SD3.5, Galaxy, 250 commits, topology_mapper.cpp:519 FATAL
6. `892948__fd218c88` — test_pipeline_motif, Motif, Galaxy, 250 commits
7. `1080529165__c96333e2` — test_qwen_accuracy, Qwen, Galaxy, 250 commits, card hang

Test commands for pending (all from candidates_merged.json authoritative paths):
- 892931: `pytest models/tt_dit/tests/models/mochi/test_vae_mochi.py::test_tt_decoder_forward -x`
- 892930: `pytest models/tt_dit/tests/models/mochi/test_vae_mochi.py::test_tt_upsample_forward -x`
- 892891: `pytest models/tt_dit/tests/models/flux1/test_transformer_flux1.py::test_single_transformer_block -x`
- 892949/892948: paths from opus-triage-2026-05-16.json
- 1080529165: path from opus-triage-2026-05-16.json

---

## Key files
- `campaign-state.json` — authoritative state for all bisects/verifications/resolved/pending
- `confirmed-escapes.json` — confirmed escapes with fix commits
- `candidates_merged.json` — raw candidate data from Snowflake (authoritative filepaths)
- `opus-triage-2026-05-16.json` — Opus analysis for 2026-05-16 candidates batch
- Confluence page: 2424012846 (MI6 space, "bug escapes")

---

## Runner labels (confirmed)
- BH-LoudBox (blackhole): `["BH-LoudBox"]`
- Galaxy wh_galaxy (wormhole_b0): `["arch-wormhole_b0", "pipeline-functional", "topology-6u"]`
- T3K wh_t3000 (wormhole_b0): `["config-t3000"]`
- N300 (wormhole_b0): `["N300"]`
- test-dispatch.yaml workflow ID: `103409066`

---

## On resume checklist
1. Check all 3 active run results (25964583087, 25964568086, 25964730548)
2. For each completed run: record result in campaign-state.json, calculate next step, dispatch if needed
3. DM Evan with results summary
4. Update Confluence page 2424012846
5. Move 867860 to final confirmed state (if not already done)
6. Check if any queued candidates can now be dispatched (T3K or Galaxy free)
