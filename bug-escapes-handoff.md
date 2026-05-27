# Bug-Escapes Workflow — Handoff Document

**Written:** 2026-05-06
**Author:** BrAIn (handing off to new agent)
**For:** New agent taking over bug-escapes work

---

## 1. What This Is

The bug-escapes pipeline is an automated system that detects **test regressions that escaped the merge gate** — i.e., tests that started consistently failing on `main` *after* a bad commit landed, but were not caught by pre-merge CI.

The system:
1. Scans CI history for jobs with N consecutive failures
2. Uses an LLM to classify each failure (real bug vs. infra noise)
3. Identifies the exact commit on `main` that introduced each regression
4. Classifies each escape by its blast radius (vertical/horizontal/cross_layer)
5. Optionally runs live verification (actually re-running before/after commits)
6. Deduplicates across runs so the same escape isn't reported twice

**Goal:** Give the team an automated early-warning system for regressions that slip through, so they can be fixed before accumulating.

---

## 2. Repositories & Branches

| Repo | Branch | What's there |
|------|--------|-------------|
| `tenstorrent/tt-auto-triage` | `ebanerjee/bug-escapes` | All pipeline scripts (phases 0–4), reusable CI workflow |
| `tenstorrent/tt-metal` | `ebanerjee/bug-escapes` | Dispatch workflow (`triage-ci.yaml`) |

**Open PRs (DO NOT MERGE YET):**
- `tenstorrent/tt-auto-triage` PR #1106 — tracks the auto-triage branch. Description has a checklist of what must be done before merge.
- `tenstorrent/tt-metal` PR (linked from #1106) — for the tt-metal dispatch workflow

---

## 3. Pipeline Architecture

```
tt-metal@ebanerjee/bug-escapes
  └── .github/workflows/triage-ci.yaml   (workflow_dispatch trigger)
          │
          ├── detect job
          │     calls: tt-auto-triage/.github/workflows/bug-escapes-ci.yaml@ebanerjee/bug-escapes
          │
          └── verify job (only if auto-verify=true)
                calls: tt-auto-triage/.github/workflows/verify-bug-escape-ci.yaml@ebanerjee/bug-escapes
```

The heavy lifting is in `tt-auto-triage`. `triage-ci.yaml` in `tt-metal` is just the entry point.

### Workflow Inputs (triage-ci.yaml dispatch)

```
test-workflows     - comma-separated workflow filenames to scan (e.g. "vllm-nightly-tests.yaml")
auto-verify        - true/false — whether to run live verification after detection
llm-backend        - "cursor" or "copilot"
max-phase          - stop after this phase (0-4); useful for debugging
lookback-days      - how many days of CI history to scan (default: 7)
incremental-days   - how many days to look for fix commits (default: 30)
fix-commit-sha     - override (for verify-only mode)
mock-verify        - true/false — dry-run verification without actually dispatching jobs
```

---

## 4. The Five Phases

### Phase 0 — Smoke Test
- Checks that the environment is sane: GitHub token present, LLM CLI available, required tools installed
- Fails fast so you don't waste tokens on a broken environment

### Phase 1 — Discovery
- Calls the GitHub API to list all workflow runs for `tenstorrent/tt-metal`
- Classifies each workflow as `merge-gate`, `pr-gate`, or `other`
- Filters down to the workflows specified in `test-workflows` input
- **Output:** list of workflow IDs to analyze

### Phase 2 — Candidate Failures
- For each discovered workflow, finds jobs with **N consecutive failures** (default N=3, configurable via `consecutive-runs`)
- Pre-filters out infra errors (e.g., `InfraErrorV1.*` failure signatures) and known flakes
- Downloads raw logs for remaining candidates
- Sends logs to LLM for classification: real test failure vs. infra noise
- Checks against **seen-candidate cache** (`seen.json`) — skips LLM if already classified
- **Output:** list of confirmed failure candidates
- **Artifact:** `bug-escapes-seen-cache` (the seen-candidate cache, to reuse next run)

### Phase 3 — Fix Points
- For each confirmed failure, bisects `main` to find the commit that fixed/introduced it
- Uses the GitHub Commits API to walk `main` history
- Looks back up to `incremental-days` days
- **Output:** for each failure: `{test_name, fail_sha, fix_sha, workflow}`

### Phase 4 — Classification & Output
- Takes phase 3 results and classifies each escape:
  - **vertical** — only one workflow affected
  - **horizontal** — multiple workflows, same layer
  - **cross_layer** — spans multiple abstraction layers
- Assigns confidence score
- Checks against **seen-escapes cache** (see §5) — deduplicates
- Writes `bug-escapes-output.json`
- If `auto-verify=true`, also writes a verify matrix (`verify-matrix.json`)
- **Artifacts:** `bug-escapes-output`, `bug-escapes-seen-escapes-cache`

### Auto-Verify Mode
When `auto-verify=true`:
- Phase 4 emits a `verify-matrix.json` listing escapes to verify
- `verify-all` matrix job (in `bug-escapes-ci.yaml`, `max-parallel: 1`) picks up the matrix
- Each leg executes `verify.sh` which dispatches a pruned tt-metal workflow run for before/after commits
- Max-parallel=1 satisfies the ≤2 concurrent constraint (one verify + one detect at most)

---

## 5. Caching Systems

There are **two separate caches**. Don't confuse them.

### 5a. Seen-Candidate Cache (`seen.json`)
- **Purpose:** Avoid re-calling the LLM for the same job failure signature
- **Key:** failure job ID / signature
- **Artifact name:** `bug-escapes-seen-cache`
- **Managed by:** Phase 2

### 5b. Seen-Escapes Cache (`seen_escapes.json`)
- **Purpose:** Prevent the same confirmed bug escape from being reported on every subsequent run
- **Key:** `fix_sha|test_name|workflow_basename`
- **TTL:** `LOOKBACK_DAYS` days
- **Artifact name:** `bug-escapes-seen-escapes-cache`
- **Managed by:** Phase 4
- **Flow:**
  1. At start of Phase 4: download prior artifact → `_restore_seen_escapes_cache()`
  2. Evict stale entries: `_evict_stale_escapes()` (entries older than TTL removed)
  3. Filter new escapes against cache
  4. Add new escapes to cache
  5. Re-upload cache artifact after run

> **Critical bug fixed (2026-05-06):** `_evict_stale_escapes()` ended with a bare `&&` list.
> When nothing was evicted (evicted=0), `[ 0 -gt 0 ]` returned 1, the function returned 1,
> and `set -euo pipefail` killed phase 4 before it could write output. Fixed by adding
> `return 0` as the last line. Commit `41b91b7` on `ebanerjee/bug-escapes`.

---

## 6. Key Files

```
tt-auto-triage/bug-escapes/
├── run.sh                    # Orchestrates phases 0–4 sequentially
├── phase0_smoke.sh           # Smoke test
├── phase1_discovery.sh       # Workflow discovery & classification
├── phase2_candidates.sh      # Failure candidate detection + LLM classification
├── phase3_fixpoints.sh       # Fix-commit bisection
├── phase4_classify.sh        # Classification, seen-escapes cache, output generation
├── verify.sh                 # Per-escape verification (dispatches tt-metal runs)
├── lib/
│   ├── common.sh             # Sources upstream auto-triage libs; sets AT_OWNER_REPO
│   └── cursor_agent.sh       # LLM backend abstraction (cursor or copilot)
└── output/                   # Runtime directory for phase outputs

tt-auto-triage/.github/workflows/
├── bug-escapes-ci.yaml       # Reusable: detect job + verify-all matrix job
└── verify-bug-escape-ci.yaml # Reusable: single escape verify job

tt-metal/.github/workflows/
└── triage-ci.yaml            # Entry point: workflow_dispatch trigger
```

### LLM Backend Abstraction
`lib/cursor_agent.sh` abstracts the LLM call. Set `LLM_BACKEND=cursor` or `LLM_BACKEND=copilot`.
The workflow passes the appropriate API key as a secret.

---

## 7. Run History

All runs are on `tenstorrent/tt-metal`, branch `ebanerjee/bug-escapes`, workflow `triage-ci.yaml`.

| Run | Date (UTC) | Status | Notes |
|-----|-----------|--------|-------|
| [25454969849](https://github.com/tenstorrent/tt-metal/actions/runs/25454969849) | 2026-05-06 18:57 | ✅ SUCCESS | **First successful run with seen-escapes cache active.** 0 escapes found on `vllm-nightly-tests.yaml` (7-day window, auto-verify=true). Bug fix validated. |
| [25445321906](https://github.com/tenstorrent/tt-metal/actions/runs/25445321906) | 2026-05-06 15:38 | ❌ FAIL | Phase 4 crash — `_evict_stale_escapes()` returned 1 when evicted=0, `set -e` fired. Fixed in commit `41b91b7`. |
| [25335335452](https://github.com/tenstorrent/tt-metal/actions/runs/25335335452) | 2026-05-04 | ❌ CANCELLED | — |
| [25332021459](https://github.com/tenstorrent/tt-metal/actions/runs/25332021459) | 2026-05-04 | ❌ CANCELLED | — |
| 25221610978–25204416620 | 2026-05-01 | ✅ Various | Hourly cron runs from earlier development |

**Last known good output (run 25454969849):**
```json
{
  "bug_escapes": [],
  "total_escapes": 0,
  "auto_verify_enabled": true,
  "verification_candidates": 0,
  "scan_window_days": 7,
  "workflows_scanned": ["vllm-nightly-tests.yaml"]
}
```
(0 escapes = no regression detected in vllm-nightly-tests in the past 7 days; this is expected / healthy)

---

## 8. How to Dispatch

Go to:
https://github.com/tenstorrent/tt-metal/actions/workflows/triage-ci.yaml

Select branch: `ebanerjee/bug-escapes`

Click "Run workflow" with inputs. Example for a normal run:
```
test-workflows: vllm-nightly-tests.yaml
auto-verify:    true
llm-backend:    copilot
lookback-days:  7
```

**Constraint:** Do not dispatch more than 2 pruned workflow runs simultaneously (i.e., don't start a new run while 2 verify legs are already running). With `max-parallel: 1` in the verify matrix, this is naturally satisfied as long as you don't dispatch two separate triage runs at once.

---

## 9. Known Issues & Remaining Work

### Bug fixes on branch (not yet applied — from paused work on `ebanerjee/auto-issues`)

These are NOT on the `ebanerjee/bug-escapes` branch yet:

1. **`_regex_extract_error()` in `detect_failures.py`**:
   - Missing pattern: `TT_THROW:[^\n]{0,300}` (Tenstorrent's custom assert macro)
   - Broad `FAILED` pattern matches GTest summary count lines (`[  FAILED  ] 5 tests`) — needs restriction

2. **`CONSECUTIVE` default**: Change from `"3"` to `"2"` in `__main__.py`

These are paused because they belong to a different branch (`ebanerjee/auto-issues`) and a different workflow (`create-issues`), not the bug-escapes pipeline.

### Remaining work before PR #1106 can merge

From the PR description checklist:
- [ ] Phase 1 condition-skipped job false positives investigated
- [ ] Cache TTL/eviction validated (seen-escapes cache — now partially done: code exists and ran successfully, but no live escapes observed yet to fully validate dedup behavior)
- [ ] End-to-end validation with a **confirmed actual bug escape found** (0 escapes so far)
- [ ] tt-metal PR (#42800 equivalent) ready to merge simultaneously

### Next logical step for new agent

**Run the pipeline against more workflows** to actually find a bug escape. Try:
```
test-workflows: nightly-regressions.yaml,model-perf-tests.yaml
```
or whatever workflows are most likely to have had recent failures on main.

Once a real escape is found, validate that:
1. The first run reports it
2. The second run deduplicates it (doesn't report it again)
3. The escape gets verified correctly via auto-verify

---

## 10. Architecture Diagram (Mermaid)

```
flowchart TD
    A[triage-ci.yaml\ntt-metal@ebanerjee/bug-escapes\nworkflow_dispatch] --> B[detect job\nbug-escapes-ci.yaml]
    B --> P0[Phase 0: Smoke]
    P0 --> P1[Phase 1: Discovery\nclassify workflows]
    P1 --> P2[Phase 2: Candidates\nN consecutive failures\nLLM classification\nseen-candidate cache]
    P2 --> P3[Phase 3: Fix Points\nbisect main for fix commit]
    P3 --> P4[Phase 4: Classify\nvertical/horizontal/cross_layer\nseen-escapes dedup]
    P4 --> OUT[bug-escapes-output.json\nbug-escapes-output artifact]
    P4 --> MATRIX[verify-matrix.json\nif auto-verify=true]
    MATRIX --> VA[verify-all matrix job\nmax-parallel=1]
    VA --> VS[verify.sh per escape\ndispatches pruned tt-metal runs]
```

---

## 11. Secrets Required

| Secret (in tt-metal) | Used for |
|---------------------|---------|
| `GITHUB_TOKEN` (built-in) | Read tt-metal CI history |
| `EVAN_CURSOR_API_KEY` | Cursor LLM backend |
| `AUTO_TRIAGE_TOKEN` | Copilot LLM backend |
| `EVAN_PAT` | Writing artifacts, dispatching verify runs |

These are configured in the `tenstorrent/tt-metal` repo settings. No action needed — they're already wired up in `triage-ci.yaml`.

---

## 12. Quick Reference — Artifact Names

| Artifact | Contents | Uploaded by |
|----------|----------|------------|
| `bug-escapes-output` | `bug-escapes-output.json` + phase intermediate files | `bug-escapes-ci.yaml` detect step |
| `bug-escapes-seen-cache` | `seen.json` (phase 2 LLM cache) | `bug-escapes-ci.yaml` detect step |
| `bug-escapes-seen-escapes-cache` | `seen_escapes.json` (phase 4 dedup cache) | `bug-escapes-ci.yaml` detect step |

These artifacts persist for 90 days and are downloaded at the start of the next run to maintain state across runs.
