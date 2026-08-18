# Closeout checklist

What Step 4 and Step 5 scan for, kept separate from `SKILL.md` because the token list grows as
`decompose-plan`'s templates grow. Sourced directly from
`plugins/decompose-plan/skills/decompose-plan/references/tasks-template.md` and
`references/handoff-template.md` — re-check both if either template changes.

## `tasks.md` is complete when

- No `(pending batch)` remains in the `Commits` column, except rows the user explicitly declined
  to place during Step 4's mapping — those stay `(pending batch)` on purpose, not by omission.
- Every `completed` phase has a `Finished` timestamp.
- No phase's `Status` is still `pending`, `in_progress`, or `blocked`.
- `Final Summary` is written — not left as the template's
  `_(Written by the coordinator once every phase reaches completed. ...)_` placeholder.
- `**Last updated:**` is today's date.

## `handoff.md` is complete when none of these survive

Literal tokens, taken from `handoff-template.md` itself, so the scan is mechanical rather than
interpretive:

```
{PLAN_NAME}  {PLAN_KEY}  {MASTER_PLAN_FILENAME}  {branch-name}  {base-branch}
{N}  {NN}  {Component 1}  {Component 2}
_(to be filled)_
_(to be filled by the coordinator after the last round)_
```

`{N}` and `{NN}` also recur inside the per-stack "How to verify" and "Recommended code review"
example blocks (`{module-1}`, `{FQCN-1}`, `{test-project-1}`, and similar) — those are stack-scoped
placeholders inside whichever example block the coordinator keeps for this project's stack; treat
any of them still present as a hit too.

State the general rule, because the templates will grow and this list will fall behind them: **any
surviving `{...}` token, and any italic parenthetical `_(...)_`, is a hit.** Scan for the pattern,
not just the examples above.

Note what this rule does *not* catch: `tasks.md`'s own empty-state markers — `_none_` under Active
blockers, `_none yet_` under Decisions — have no parentheses inside the underscores and are a
legitimate "nothing to report" value, not a placeholder left behind. Don't flag those.
