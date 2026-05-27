# Bug Escapes Campaign — Agent Handoff

This repo contains every markdown document from BrAIn (the AI SW team assistant at Tenstorrent)
related to the **bug escapes verification campaign** for `tenstorrent/tt-metal`.

The campaign goal: find test regressions that merged to `main` undetected — i.e., a test that
was passing, then failing, with a clear causal fix commit — and classify/report them.

---

## Start Here

1. **`bug-escapes-data/RUNBOOK.md`** — the authoritative procedure. Read this first, every session.
   VERIFICATION-RUNBOOK.md is deprecated; ignore it.
2. **`bug-escapes-data/RESUME.md`** — quick-start context for picking up mid-campaign.
3. **State files** (not in this repo — live on BrAIn's filesystem):
   - `/workspace/group/bug-escapes-data/confirmed-escapes.json` — source of truth for all escape statuses
   - `/workspace/group/bug-escapes-data/seen-escapes.json` — full list of all candidates processed

---

## How BrAIn Manages Context Resets (MEMORY.md Protocol)

Claude agents have finite context windows. When the context compresses, BrAIn loses working
memory. Here's the protocol BrAIn wrote into its persistent `MEMORY.md` to survive resets:

### What MEMORY.md contains (relevant excerpts)

```markdown
## Context Reset Protocol

When you see the "Summary" block at the top of a conversation, your context was compressed.

1. Send Evan a reset notification via send_message:
   "⚠️ My context was compressed and I've been reset."

2. Do NOT automatically re-read the runbook or resume campaign work.
   Wait for Evan to explicitly tell you to resume the bug escapes campaign
   before reading any campaign files or dispatching any probes.

Campaign work resumes only on explicit instruction from Evan.

## Bug Escapes Campaign — MANDATORY SESSION START

**⚠️ When resuming campaign work: READ THE RUNBOOK FIRST.**
`cat /workspace/group/bug-escapes-data/RUNBOOK.md` before doing anything else.
Memory and summaries are lossy. The runbook is the source of truth for procedure.
Do not rely on remembered rules — verify against the runbook every session.

## Escape ID Format — CRITICAL

`{test_case_id}__{last_failing_sha[:8]}` — always `last_failing_sha`, NEVER `first_pass_sha`.

## Escape Confirmation — CRITICAL

No escape can be stamped `confirmed` without BEFORE+AFTER dispatch probes.
Static file-path / git log analysis alone is NOT confirmation. It is a candidate.

## Bug Escapes Campaign

State lives in external files — do NOT store escape counts or status here.

- State file: `/workspace/group/bug-escapes-data/confirmed-escapes.json` (source of truth)
- Runbook: `/workspace/group/bug-escapes-data/RUNBOOK.md`

**⚠️ DURING campaign work — write state continuously:**
- After every escape confirmed/refuted/bisect-step: update confirmed-escapes.json immediately
- Each escape must have explicit "phase" field: regression_found / bisect_in_progress /
  awaiting_verification / confirmed / refuted
- Do NOT rely on context memory as source of truth — the JSON file is the source of truth
- After every 3 escapes processed: update Confluence page 2424012846
```

### Key design decisions behind this protocol

**Problem**: Claude's context compresses mid-campaign. The compressed "Summary" block is lossy —
it drops nuance, fabricates details, and misremembers escape IDs. Acting on it without re-reading
the runbook leads to mistakes (wrong escape IDs, wrong statuses, wrong dispatch decisions).

**Solution — three layers of persistence:**

1. **MEMORY.md** (loaded every session automatically): Contains _rules_, not _state_. Rules are
   short and stable. State is not. MEMORY.md tells the agent WHERE to find state, not what the
   state is. It also contains the "read the runbook first" mandate, so even after a full context
   loss the agent knows to re-anchor before acting.

2. **RUNBOOK.md** (read at campaign start): The full procedure — every step, every edge case,
   every lesson learned from past mistakes. This is the recovery document. If the agent has
   forgotten everything, RUNBOOK.md is the single file that lets it reconstruct enough context
   to work safely.

3. **confirmed-escapes.json** (updated after every action): Append-only state log. Never trust
   the agent's memory of "what we processed" — always re-read the JSON. This prevents double-
   processing, status drift, and the agent acting on hallucinated state.

**What NOT to put in MEMORY.md:**
- Escape counts ("we've confirmed 7 escapes") — these go stale immediately
- File paths with line numbers — these change
- Escape IDs or SHAs — too easy to hallucinate subtly wrong values
- Anything that should be in the runbook — MEMORY.md is an index, not a copy

**The reset notification rule**: When the agent detects a context reset (Summary block at top),
it must notify Evan _before_ resuming any work. This prevents silent continuation on stale/wrong
context. Evan can then decide whether to resume, re-brief, or redirect.

---

## Key Technical Facts (for the new agent)

### Infrastructure
- **Repo**: `tenstorrent/tt-metal`
- **CI data**: Snowflake `TTDATASF.SW_TEST` (read-only)
- **Confluence**: pages 2424012846 (full list) and 2428895268 (confirmed table)
- **GitHub token**: available as `$GITHUB_TOKEN` in BrAIn's container
- **No `gh` CLI** — use raw `curl` against GitHub REST/GraphQL APIs

### Snowflake gotchas
- Column is `H.HOST_NAME` not `H.HOSTNAME`
- Column is `J.JOB_SUCCESS` not `J.SUCCESS`
- CICD_TEST has 5.8B rows — ALWAYS filter by date and use LIMIT
- `CICD_PIPELINE.PROJECT` = `'tt-metal'` (not `tenstorrent/tt-metal`)
- `CICD_PIPELINE.GIT_COMMIT_HASH` (not `GIT_SHA`)
- LLK assert nightly runs (`J.NAME LIKE '%LLK asserts%'`) are a special build variant —
  exclude from island analysis; they run on different cadence and different conditions

### Escape classification rules
- **Escape ID**: `{test_case_id}__{last_failing_sha[:8]}` — ALWAYS last_failing_sha
- **Island**: contiguous block on main branch with 0 passes on single-card boards only
  (N150/N300/P150/P100/P150b — NOT t3k/galaxy/TG)
- **Horizontal**: fix_layer == test_layer
- **Vertical**: fix_layer < test_layer
- **Layer hierarchy**: L1=tt_metal/hw+llk, L2=tt_metal/impl+api, L3=ttnn, L4=models
- **Fabricated SHA detection**: if supposed `last_failing_sha` is AFTER the fix commit
  in topological order, the entry is fabricated — mark as `skipped_flap` and document why
- **Flap**: same SHA has both PASS and FAIL, or no contiguous 0-pass block → `skipped_flap`
- **Confirmation requires dispatch probes**: static analysis alone is NOT confirmation

### Probe dispatch rules (critical)
- Artifact reuse requires ALL THREE inputs: `dev-docker-image`, `build-artifact-name`,
  `wheel-artifact-name` — passing only some causes silent wrong behavior
- Merge gate artifacts are Release builds — `build-artifact-name` will NOT contain `_profiler_`
- NEVER dispatch the full CI workflow — prune to target test job + its `needs:` chain only
- Check `J.NAME` of original failing job — if "LLK asserts" or "watcher", probe must use
  the same build variant or result is invalid

---

## File Index

| File | Description |
|------|-------------|
| `bug-escapes-data/RUNBOOK.md` | **Primary procedure** — read first every session |
| `bug-escapes-data/RESUME.md` | Quick-start context summary |
| `bug-escapes-data/README.md` | Data directory overview |
| `bug-escapes-data/postmortem-2026-05-16.md` | Post-mortem on the fabricated-SHA bug (read this) |
| `bug-escapes-data/campaign-analysis-2026-05-11.md` | Mid-campaign analysis |
| `bug-escapes-data/evan-proposal-analysis.md` | Analysis of Evan's original proposal |
| `bug-escapes-data/VERIFICATION-RUNBOOK.md` | **Deprecated** — superseded by RUNBOOK.md |
| `bug-escapes-findings.md` | Findings log |
| `bug-escapes-handoff.md` | Earlier handoff doc (pre-dates this file) |
| `bug-escapes-plan.md` | Original campaign plan |
| `bug-escapes-roadmap.md` | Feature roadmap |
| `bug-escapes-week-plan.md` | Week-level planning |
| `bug_escapes_source.md` | Source data notes |
| `cleanup-bug-escapes.md` | Cleanup tasks |
| `verification_report.md` | Verification report |
