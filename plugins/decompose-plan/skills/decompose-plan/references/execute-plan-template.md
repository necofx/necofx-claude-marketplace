# Execute Plan — {PLAN_NAME}

This file is the **round-by-round orchestrator prompt** for the multi-agent (teammate) execution of the decomposed plan in this folder.

**How to use:** open a fresh conversation, paste the entire "Coordinator Prompt" block below, and run it. The coordinator dispatches teammates one round at a time and advances when every phase in the round is `completed`. Commits are **deferred to batches** — the coordinator surfaces a batched `git add … && git commit -m "…"` block at the end of each round (or at end-of-plan, depending on user preference); phases never request their own commits.

**Folder:** `{FOLDER}`
**Team name:** `{PLAN_SLUG}`
**Master plan:** `{FOLDER}/{MASTER_PLAN_FILENAME}`

## Rounds in this plan

### Round 0 (sequential — 1 teammate)
- `phases/PHASE-00-{slug}.md` — {Phase 00 Title} — owner: `{owner-agent}`

### Round 1 (parallel — N teammates, no shared files)
- `phases/PHASE-01-{slug}.md` — {Phase 01 Title} — owner: `{owner-agent}`
- `phases/PHASE-02-{slug}.md` — {Phase 02 Title} — owner: `{owner-agent}`
- `phases/PHASE-03-{slug}.md` — {Phase 03 Title} — owner: `{owner-agent}`

### Round 2 (sequential — 1 teammate)
- `phases/PHASE-04-{slug}.md` — {Phase 04 Title} — owner: `{owner-agent}`

### Round 3 (sequential — 1 teammate)
- `phases/PHASE-05-{slug}.md` — {Phase 05 Title} — owner: `{owner-agent}`

---

## Coordinator Prompt

```
You are the coordinator for the implementation of "{PLAN_NAME}".

The plan has been decomposed into atomic phase files in this folder:
{FOLDER}

The decomposition is organized in ROUNDS:
- Phases in the SAME round run in PARALLEL (dispatch all in a single message).
- Rounds run SEQUENTIALLY. Between rounds, you summarize and either (a)
  surface a batched commit command for the user to run, or (b) continue
  straight into the next round if the user prefers a single end-of-plan
  batch. Phases NEVER request their own commits — Claude Code is read-only
  for git per `.claude/rules/git-workflow.md`, but the friction of one
  commit-request per phase is unacceptable for multi-agent flow, so all
  commits are batched by the coordinator.

## Setup (do once, before Round 0)

1. Invoke `Skill(skill="superpowers:using-superpowers")` — skill discipline.
2. Invoke `Skill(skill="superpowers:subagent-driven-development")` — dispatch
   discipline.
3. Invoke `Skill(skill="superpowers:dispatching-parallel-agents")` if a round
   has more than one phase.
4. Read `{FOLDER}/{MASTER_PLAN_FILENAME}` end-to-end.
5. Read `{FOLDER}/tasks.md` — single source of truth for status.
6. Read every `{FOLDER}/phases/PHASE-*.md` so you know goals, dependencies,
   owner-agent, and acceptance criteria across the whole plan.
7. Print a one-paragraph summary of what you're about to run, including the
   round structure and the file-conflict-matrix verdict.

## Per-round loop

Repeat until all rounds are completed.

### A. Pick the next round

The next round is the lowest-numbered round where at least one phase has
status `pending`. If every phase in the current round is `completed`, advance
to the next round.

### B. Dispatch every ready phase in the round IN PARALLEL

A phase is ready when:
- Its status is `pending`, AND
- All phases in earlier rounds (and any explicit `Dependencies` it names)
  have status `completed`.

In a SINGLE message, make one Agent tool call per ready phase using this
shape:

  Agent(
    subagent_type="{owner-agent}",         # from the phase's "Owner Agent" line
    name="phase-{NN}",                      # zero-padded
    team_name="{PLAN_SLUG}",
    description="Execute phase {NN}: {Phase Title}",
    prompt="""
    You are teammate `phase-{NN}` working on phase {NN} of "{PLAN_NAME}".

    Your single source of truth is the phase file:
      {FOLDER}/phases/PHASE-{NN}-{slug}.md

    Workflow:
    1. Read your phase file in full.
    2. Invoke EVERY skill listed under "Skills to Invoke (teammate-side)" via
       the Skill tool, in the order listed. Start with
       `superpowers:using-superpowers`.
    3. Read EVERY file listed under "Documents to Read". If a file is missing,
       report it back in your tasks.md per-phase notes and continue.
    4. Perform the Pre-execution check: claim Phase {NN} in `{FOLDER}/tasks.md`
       (set status=in_progress, fill Agent + Started). DO NOT request a
       commit for the claim — commits are batched by the coordinator.
    5. Execute every Atomic Step in order. **Do NOT request commits between
       steps.** Keep your working tree clean and tested; the coordinator will
       surface a batched commit at end-of-round or end-of-plan.
    6. Verify EVERY item in "Verification". Use the
       `superpowers:verification-before-completion` skill before claiming
       done.
    7. When (and only when) every Verification item is green, set your row in
       tasks.md to `Status = completed`, `Commits = (pending batch)`, and
       append a final summary entry.
    8. If you hit a blocker you cannot resolve, set Status = blocked, fill the
       Active blockers section of tasks.md with the blocker, and return
       control to me with a one-paragraph summary.

    Hard rules:
    - Only edit Phase {NN}'s row and your own Detailed Progress section in
      tasks.md. Never touch other phases' rows or sections.
    - Do not start phases other than your own.
    - Follow CLAUDE.md and every `.claude/rules/*.md` file in this repo.
    - You are read-only for git. Do NOT request commits. The coordinator
      batches commits at end-of-round or end-of-plan and surfaces them to
      the user.

    Report back when you finish (or get blocked) with a one-paragraph summary
    of what landed and any deviations from the plan.
    """
  )

If there is only one ready phase in the round, dispatch it alone — but keep
the same `name`/`team_name`/`subagent_type` shape so the coordination model
stays uniform.

### C. Monitor running teammates

Between dispatches and during waits, re-read `{FOLDER}/tasks.md` to track
status. Use `SendMessage(to="phase-{NN}", message="…")` to ask a still-running
teammate for an update on tasks.md or a clarification.

If a teammate goes silent for an unreasonable time, send them a check-in
message asking them to summarise progress and update tasks.md.

### D. End-of-round summary (mandatory)

When every phase in the current round has reached `completed` (or `blocked`
/ `dropped`):

1. Re-read `{FOLDER}/tasks.md` to confirm.
2. Read every Detailed Progress section for the round's phases — flag any
   deviation from the plan that downstream phases need to know about.
3. Run the project's full build + test gate (from the matching tech-stack
   profile) to confirm cross-phase integration is clean. If it fails, dispatch
   a fix teammate or pause for user input — do NOT advance to the next round
   on a broken integration.
4. Append a `### Round {R} summary` block to the `## Coordination Notes`
   section of tasks.md with: phases completed, key deviations, file
   coordination outcomes, anything Round {R+1} teammates should know.
5. Assemble the **batched commit command** for the user covering every file
   touched in this round (use the project's commit format). Present it as a
   copy-pasteable block and tell the user it is optional — they may run it
   now, defer to end-of-plan, or batch it with later rounds at their
   discretion. **Do NOT block on the user committing it.**
6. Advance to the next round unless the user has signalled otherwise (a
   `/goal` directive, a manual pause, an unresolved blocker). The default
   behaviour is autonomous round progression.

## Handling blockers

When a phase reports `blocked`:
- Read the blocker entry in `## Active blockers` and the phase's Detailed
  Progress section.
- Decide: resolve manually, re-dispatch with new context, drop the phase, or
  pause the whole round.
- Record the decision in `## Decisions` of tasks.md.
- If resolving, dispatch a fresh teammate with the same name + team but a
  prompt that names the blocker and the resolution.

## Completion

When the last round completes:

1. Re-read every Detailed Progress section in `tasks.md`.
2. Write the `## Final Summary` block at the bottom of `tasks.md`:
   - What was delivered against the master plan
   - Phase count, test count, total commits (after batches land), PRs
   - Deviations from the master plan
   - Open items
3. Populate `{FOLDER}/handoff.md` (the scaffold exists; fill every section).
   `handoff.md` is the artifact your reviewer reads first.
4. Assemble the **final batched commit command(s)** — one per logical group
   (per round, per layer, or one mega-commit, depending on user preference).
   Default: one commit per round so history is reviewable but not noisy.
   Present each as a copy-pasteable block. Each commit covers exactly the
   files touched in the corresponding round, plus the relevant tasks.md
   row updates.
5. Report back to the user with:
   - A one-paragraph summary of what was built
   - The path to `tasks.md` and `handoff.md`
   - The batched commit commands ready to copy-paste
   - The recommended next step (e.g. "Run /code-review against this branch",
     "Run the Operator manual-verification checklist in MASTER_PLAN.md").
```

---

## Tips for the coordinator (out-of-band reminders)

- **Parallelism is the whole point.** When a round has N phases, ALL N must be dispatched in one message containing N Agent tool calls. Sequential dispatch wastes wall-clock time.
- **Commits are batched, not per-phase.** Phases NEVER request their own commits. You assemble batched commits at end-of-round (default) or end-of-plan (when the user prefers). Make commit commands copy-pasteable (use back-tick line continuation on PowerShell, `\` on bash). The user may run them whenever they wish; do not block on them.
- **Cadence.** The Agent harness notifies you when a teammate completes — don't sit in a sleep loop. If you must poll because of out-of-band external state, prefer one long poll over many short ones.
- **Scope discipline.** Teammates work only on their phase. If a teammate proposes cross-phase work, capture it in `## Coordination Notes` and decide as coordinator whether to spawn a new phase or fold the work into an existing one.
- **End-of-plan verification.** Before flipping the final phase to `completed`, re-read the master plan and confirm every section is reflected in delivered work. If something is missing, open a follow-up phase rather than silently dropping it.
