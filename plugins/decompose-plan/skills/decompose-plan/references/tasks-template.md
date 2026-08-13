# {PLAN_NAME} — Task Tracker

**Single source of truth for phase progress.** Every teammate MUST update this file at two points (on pickup, on completion) — see § "Update protocol" below. Commits are batched by the coordinator; teammates never request commits themselves.

**Last updated:** {YYYY-MM-DD} — by coordinator (plan decomposition)
**Decomposition strategy:** {one line — e.g. "by architectural layer", "vertical feature slice", "risk-first (spike → fan-out → integration)"}
**Total phases:** {N} across {R} rounds
**Team name:** `{PLAN_SLUG}`
**Coordinator:** {agent-name-or-user}

---

## Phase status

| # | Phase | Round | Owner agent | Status | Agent | Started | Finished | Commits | PR |
|---|---|---|---|---|---|---|---|---|---|
| 00 | {Phase 00 Title} | 0 | {owner-agent} | pending | — | — | — | — | — |
| 01 | {Phase 01 Title} | 1 | {owner-agent} | pending | — | — | — | — | — |
| 02 | {Phase 02 Title} | 1 | {owner-agent} | pending | — | — | — | — | — |
| 03 | {Phase 03 Title} | 1 | {owner-agent} | pending | — | — | — | — | — |
| 04 | {Phase 04 Title} | 2 | {owner-agent} | pending | — | — | — | — | — |
| 05 | {Phase 05 Title} | 3 | {owner-agent} | pending | — | — | — | — | — |

**Status legend:** `pending` · `in_progress` · `blocked` · `completed` · `dropped`

## Rounds + dependencies

```
Round 0 (sequential — 1 teammate)
    Phase 00 — {Foundation phase title}
        |
        v
Round 1 (parallel — 3 teammates, no shared files)
    Phase 01: {…}
    Phase 02: {…}
    Phase 03: {…}
        |    (all three must complete before Round 2 starts)
        v
Round 2 (sequential — 1 teammate)
    Phase 04 — {Integration phase title}
        |
        v
Round 3 (sequential — 1 teammate)
    Phase 05 — {Verification phase title}
```

**Wall-clock estimate:** ~{X}h sequential; ~{Y}h with Round 1 fully parallelised.

## File-conflict matrix (parallel rounds)

Round 1 file-conflict check — confirm before dispatch:

| File | Phase 01 | Phase 02 | Phase 03 |
|---|---|---|---|
| `path/to/shared/file.cs` | Modify | — | — |
| `path/to/other/file.cs` | — | Modify | — |

{If any cell shows two "Modify" entries in the same row, document the coordination rule below.}

---

## Update protocol

### When you (the teammate) START a phase

1. Open this file (`tasks.md`) in your editor / Edit tool.
2. Change the phase's `Status` from `pending` to `in_progress`.
3. Fill the `Agent` column with your subagent name (e.g. `phase-01-dotnet-senior-developer`) and `Started` with `YYYY-MM-DD HH:MM` (24h, machine-local time).
4. Append a one-line entry under your Detailed Progress section: `- YYYY-MM-DD HH:MM — picked up`.
5. **Do NOT request a commit for the claim.** Commits are batched by the coordinator at end-of-round or end-of-plan. Just edit tasks.md and continue with your work.

### When you FINISH the phase

1. Change `Status` to `completed`.
2. Fill `Finished` with `YYYY-MM-DD HH:MM`.
3. Set `Commits` to `(pending batch)` — the coordinator fills in the real SHA(s) after the batched commit lands.
4. If a PR was opened (after batches), paste the URL or PR# in the `PR` column.
5. Append a final summary entry to your Detailed Progress section: deliverables, test count, deviations.
6. Hand back to the coordinator. **Do NOT request a commit.**

### When the coordinator LANDS A BATCH commit covering your phase

(Coordinator-only — teammates don't do this themselves.)

1. Replace `(pending batch)` in your `Commits` column with the short SHA(s) (7 chars), comma-separated.
2. Append a one-line entry under the phase's Detailed Progress section describing what the batch landed and which other phases it covered.

### When you are BLOCKED

1. Change `Status` to `blocked`.
2. Add an entry to **Active blockers** below with: phase #, blocker summary, who needs to resolve.
3. Hand back to the coordinator with a one-paragraph explanation.

### When the phase is DROPPED (decision not to do it)

1. Change `Status` to `dropped`.
2. Add a justification line under **Decisions** below.

### Hard rules

- Edit ONLY your own row and your own Detailed Progress section. Never touch other phases' rows or sections.
- Phases NEVER request commits. The coordinator batches commits at end-of-round or end-of-plan and surfaces them to the user.
- When the coordinator does assemble a batch, never use `git add -A` from this folder — stage `tasks.md` plus the phase's actual changes explicitly.
- Date/time format is always absolute (`YYYY-MM-DD HH:MM`), never relative ("today", "now", "Thursday").

---

## Detailed Progress

### Phase 00 — {Phase 00 Title}
- _(updates appended by phase-00 teammate)_

### Phase 01 — {Phase 01 Title}
- _(updates appended by phase-01 teammate)_

### Phase 02 — {Phase 02 Title}
- _(updates appended by phase-02 teammate)_

### Phase 03 — {Phase 03 Title}
- _(updates appended by phase-03 teammate)_

### Phase 04 — {Phase 04 Title}
- _(updates appended by phase-04 teammate)_

### Phase 05 — {Phase 05 Title}
- _(updates appended by phase-05 teammate)_

---

## Active blockers

_none_

## Decisions

_none yet_

## Coordination Notes

Coordinator-only section. Round summaries, cross-phase decisions, file-conflict resolutions, reassignments, scope adjustments.

- _(none yet)_

---

## Final Summary

_(Written by the coordinator once every phase reaches `completed`. Summarise what was delivered, deviations from the master plan, open items, and link to `handoff.md` for code review.)_
