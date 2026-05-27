# Cleanup Bug Escapes — Verification Runbook

## Instructions (from Evan, 2026-05-20)

Goal: verify or disprove every unconfirmed bug escape on:
https://tenstorrent.atlassian.net/wiki/spaces/MI6/pages/2428895268/Vertical+Bug+Escapes+Confirmed+Table

## Rules

1. **2 escapes at a time** = 4 parallel runs (1 parent + 1 fix per escape)
2. **Single-card jobs first**, T3K and Galaxy last
3. Each run = pruned workflow with only the single failing job
4. **After every pair of dispatches**: update BOTH this file AND Confluence page 2428895268
5. **If parent passes OR fix fails**: escape is NOT real → remove from Confluence immediately
6. **If parent fails AND fix passes**: escape confirmed → update Confluence with run links
7. **NEVER rebuild artifacts** — always reuse pre-built artifacts from the merge gate run that landed the commit. Rebuilding wastes ~30 min per run. See "Artifact reuse" section below.

## Artifact reuse (MANDATORY)

Never let a probe run rebuild tt-metal or the Python wheel. Each commit that landed via the merge queue has a pre-built artifact set from its merge gate CI run.

### Finding merge gate artifacts for a commit SHA

```bash
SHA=<fix_commit or fix_commit^>
TOKEN=$(env | grep GITHUB_TOKEN | cut -d= -f2)

# 1. Find the merge gate workflow run for this SHA
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/tenstorrent/tt-metal/actions/runs?head_sha=$SHA&per_page=50" \
  | jq '.workflow_runs[] | select(.name | test("merge queue|post-commit"; "i")) | {id, name, status, conclusion}'

# 2. Get artifacts from that run (look for build and wheel artifacts)
RUN_ID=<merge_gate_run_id>
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://api.github.com/repos/tenstorrent/tt-metal/actions/runs/$RUN_ID/artifacts" \
  | jq '.artifacts[] | {name, id, size_in_bytes}'
```

Key artifact names to look for:
- `build-artifact-name`: typically `TTMetal_build_<arch>` or similar
- `wheel-artifact-name`: typically `PyWHL_<something>`

### Passing artifacts to skip rebuild

`blackhole-post-commit.yaml` accepts optional inputs `dev-docker-image`, `build-artifact-name`, and `wheel-artifact-name` — when provided, the build step is skipped. This is the target workflow for BH single-card probes.

The dispatch wrapper on each `brain/` branch must call `blackhole-post-commit.yaml` via `workflow_call` with these artifact inputs populated from the merge gate run.

## Workflow dispatch

- Parent probe: dispatch on `fix_commit^` (commit just before fix), expect FAIL
- Fix probe: dispatch on `fix_commit`, expect PASS
- Workflow: `tenstorrent/tt-metal/.github/workflows/triage-ci.yaml`
- Branch pattern for dispatch: `brain/escape-before-{escape_id}` and `brain/escape-after-{escape_id}`

## ⚠️ MANDATORY: Prune the workflow before dispatching

**NEVER dispatch the full `blackhole-post-commit.yaml` workflow.** It has 47–51 jobs and wastes CI resources. You must prune it to only the single job under test.

### How to prune

1. Check out the probe branch (e.g. `brain/escape-before-890577__3ef40fe1`)
2. Open `.github/workflows/blackhole-post-commit.yaml`
3. Find the specific job name that matches the failing test (e.g. `blackhole deepseek per-core allocation tests (fast dispatch)`)
4. Delete ALL other jobs from the `jobs:` block except:
   - The target test job
   - Any jobs that job explicitly `needs:` (typically just `build-artifact`)
5. Commit and push — the dispatch will then only run those jobs
6. Verify job count before dispatching: should be 2–3 jobs max (build + 1–2 test jobs)

## State: Verified (have probe runs already on Confluence)

| Escape ID | Parent run | Fix run | Status |
|-----------|-----------|---------|--------|
| 25458544__a9a457b3 | 25710791347 FAIL | 25775757192 PASS | ✅ CONFIRMED |
| 892877__9d4ab66e | 25975346798 FAIL | 25975615675 PASS | ✅ CONFIRMED |
| 886715__438eaf0d | 26125205424 FAIL | 26117677376 PASS | ✅ CONFIRMED |
| 882415__d5da685c | 26130245428 FAIL | 26130241107 PASS | ✅ CONFIRMED |
| 892871__f8f3d4109f | 26130351088 FAIL | 26130347583 PASS | ✅ CONFIRMED |
| 867860__c5268e19 | ⬜ (bisect only) | 25950477846 PASS | ✅ CONFIRMED (partial) |
| 892929__ee56bd21 | ⬜ (bisect only) | ? | needs parent probe |

## State: Queue (unverified, single-card first)

### Batch 1 (in progress)
- TBD

### Pending single-card escapes
- 890577__3ef40fe1 (fix_layer=1, test_layer=4, PR #42611)
- 888534__9d4ab66e (fix_layer=3, test_layer=4, PR #42426)
- 884766__a5567565 (fix_layer=L1, test_layer=L4, PR #44545)
- 1080529165__fa2981b0 (fix_layer=L1, test_layer=L4, PR #44085)
- 893095__580cd8e6 (fix_layer=L1, test_layer=L4, no fix_pr)
- 879678__b93a3a3a (fix_layer=L2, test_layer=L3, no fix_pr)
- f622a01ede61 cluster (12 escapes, L1→L3, PR #44365)
- 8e823c23 cluster (2 escapes, L2→L4, PR #44621)
- e3459baf pair (2 escapes, L3→L4, PR #44557)
- 1581453070__8e823c23 (fix_layer=L2, test_layer=L4, PR #44621)
- 1639999712__8e823c23 (fix_layer=L2, test_layer=L3, PR #44621)

### Pending T3K/Galaxy escapes
- (identify from Confluence page)

## Last updated
2026-05-20 17:02Z — All 3 active runs still in progress (no failures yet). 890577 fix probe 26176360097: models-unit-tests PASS, ops/ttnn still queued. 8e823c23 parent 26176364005: most jobs still queued. 8e823c23 fix 26176367706: tt-cnn/ttnn/umd jobs in_progress. Next check: hourly cron will record results once complete.

## Previous last updated
2026-05-20 16:42Z — Cancelled duplicate 16:17 runs, redispatched 3 probes at 16:38. Active runs: 890577 fix → 26176360097, 8e823c23 parent → 26176364005, 8e823c23 fix → 26176367706. Root cause: tracy fix commits pushed after initial dispatches.

## 8e823c23 cluster (escapes 1639999712 + 1581453070)
- fix_commit: 43d283e02818bba72cd8abd44cae244df2f718fa
- parent: 215668f33857a78632b1b53ca7eb3c43ae84c22e
- Merge gate run for fix: 26087214024
- Merge gate run for parent: 26086481715
- Fix probe run v1: 26173740333 FAILED (infra — tracy=true but no profiler artifact in merge gate)
- Parent probe run v1: 26173746839 CANCELLED
- Fix probe run v2: 26175268186 CANCELLED (duplicate)
- Parent probe run v2: 26175271942 CANCELLED (duplicate)
- Fix probe run v3: https://github.com/tenstorrent/tt-metal/actions/runs/26176367706 (queued)
- Parent probe run v3: https://github.com/tenstorrent/tt-metal/actions/runs/26176364005 (queued)
- Redispatched 2026-05-20 16:38Z

## 890577__3ef40fe1 (escape 890577)
- test: test_rmsnorm (models/demos/deepseek_v3_b1/tests/unit_tests/test_rmsnorm.py)
- fix_commit: cd1b6a0167498bf7e79fbb3eeb10362066a921f8 (PR #42611 "Update attention to respect per core allocator")
- parent: 31885021a18775990ff906f4d8424bbad16e3985
- Merge gate run for fix: 24645836034
- Parent probe run: 26125861829 FAIL ✅ (test_lm_head_sampling same _calculate_fill_ compile error)
- Fix probe run v1: 26174569730 CANCELLED (same tracy=true issue)
- Fix probe run v2: 26175276649 CANCELLED (duplicate)
- Fix probe run v3: https://github.com/tenstorrent/tt-metal/actions/runs/26176360097 (queued)
- Redispatched 2026-05-20 16:38Z
