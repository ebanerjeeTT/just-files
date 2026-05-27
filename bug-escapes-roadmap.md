# Bug Escapes Improvement Roadmap

## Shipped (merged to ebanerjee/bug-escapes)

### Q4 — Pre-mark fix (phase2_candidates.sh)
Pre-mark loop moved inside the `if cursor_agent_from_template...then` success branch.
Previously a LLM timeout would suppress real failures for up to 48h (EXACT_KEY_TTL_HOURS).
Now candidates are only marked `infra_noise` after a successful LLM response.

### L1a — Enriched Phase 3 commit context (phase3_fixpoints.sh + find_fix_commit.txt)
For each commit in the transition window, Phase 3 now fetches:
- Changed file list (up to 20 files for small windows, 15/10 for medium/large)
- PR title + body excerpt (up to 500/200/100 chars for small/medium/large windows)
- Copilot `pull-request-reviewer[bot]` review overview (up to 300/150/0 chars)

Dynamic truncation caps kick in at >30 and >60 commits to keep prompt manageable.
Prompt updated: file paths used for pass-1 elimination, PR descriptions + Copilot overview
used in pass-2 candidate scoring, confidence calibration updated.

---

## Pending

### M1 — Parallel log downloads in Phase 2
Current: Phase 2 downloads logs sequentially (one job at a time).
Plan: Use bash background jobs + `wait` to download N logs in parallel.
Expected gain: 2-4x speedup on Phase 2 wall time.
Constraint: Don't exceed GH API rate limits — cap parallelism at ~10.

### L1b — Interactive agent with on-demand file diffs
After L1a is proven in production, give Phase 3's LLM the option to request
full file diffs for specific commits (not just the file list). Agent decides
which diffs to fetch based on initial enriched context.
Depends on: L1a being stable.

### L1c — Error-to-code pre-filtering
Before passing commits to the LLM, filter by layer match:
parse the failure signature to determine which layer's code was involved,
then exclude commits whose files are entirely in other layers.
This reduces the commit list the LLM has to reason over.

---

## Architecture Notes

- Binary search (dispatching CI runs to bisect) is infeasible — too few machines
- LLM attribution is the only viable fix-finding approach
- L1b/L1c enrich the LLM's context rather than replacing the LLM step
- Verify runs (auto-verify=true) are the final correctness gate — run sparingly
- Repos: tt-auto-triage (logic), tt-metal (workflow dispatch)
- Branches: ebanerjee/bug-escapes (stable), ebanerjee/bug-escapes-experimental (dev)
