# BrAIn Memory

## Bug Escapes Campaign

### Confluence Pages (all 5)
| ID | Title | Purpose |
|----|-------|---------|
| 2424012846 | TT-Metal Vertical Bug Escapes | Vertical tracking (main list) |
| 2428895268 | Vertical Bug Escapes — Confirmed Table | Detailed confirmed vertical table |
| 2461597780 | Horizontal Bug Escapes | Horizontal tracking (main list) |
| 2460254399 | Confirmed Horizontal Bug Escapes | Detailed confirmed horizontal table |
| 2440298505 | Bug Escape Chart | 4×4 matrix visualization |

### State Files
- RUNBOOK: `/workspace/group/bug-escapes-data/RUNBOOK.md` (version 2.0, 2026-05-27, authoritative)
- campaign-state: `/workspace/group/bug-escapes-data/campaign-state.json`
- seen-escapes: `/workspace/group/bug-escapes-data/seen-escapes.json`
- confirmed-escapes: `/workspace/group/bug-escapes-data/confirmed-escapes.json`

### Context Reset Protocol
On context reset: (1) Read MEMORY.md, (2) Read RUNBOOK.md, (3) Read campaign-state.json, (4) Poll active GitHub run IDs, (5) Complete pending Confluence/DM notifications, (6) Resume.

### Campaign Status (as of 2026-05-27)
- 32 vertical confirmed + 1 horizontal confirmed (868311__97edd62e, L4→L4)
- 4 refuted, 3 blacklisted_env
- 1 backlog: 868317__ee56bd21 (simple_vision_demo, attempt #4 run 26404430245 ⏳)
- Setup complete. Awaiting Evan's start signal for 90-day backfill campaign.

### Key Constraints
- MANDATORY after every workflow_dispatch: (1) Update Confluence 2424012846 or 2461597780 with run ID+URL+⏳, (2) DM @ebanerjeeTT
- Hardware: single-card (N150/N300/P150/P300) = BEFORE+AFTER parallel; multi-card (T3K/Galaxy) = strictly sequential, 1 at a time
- Subagents: may NOT dispatch, commit, or push. Only main BrAIn session dispatches.
- GitHub: `$GITHUB_TOKEN` set, authenticated as ebanerjeeTT. `gh` CLI available.
- Escape ID format: `{test_case_id}__{last_failing_sha[:8]}`
- Layer hierarchy: L1=hw/llk/umd, L2=impl/api/fabric, L3=ttnn, L4=models
