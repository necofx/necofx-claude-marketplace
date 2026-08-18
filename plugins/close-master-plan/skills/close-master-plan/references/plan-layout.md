# Plan layout

This is the layout both `create-master-plan` and `close-master-plan` read and write. Treat it as
the single source of truth for where plan folders live, what their status header looks like, and
which file in a closed folder is authoritative.

## Directory tree

```
<plans-root>/               # docs/plans/ by default
  INDEX.md                  # closed plans only, one line each
  active/
    GH-500/                 # a plan being worked
  closed/
    GH-412/                 # completed
    GH-388/                 # superseded by GH-412
```

`<plans-root>` is a root, not a per-ticket pattern — it defaults to `docs/plans/`.

## Resolving `<plans-root>`

Read the configured plan-folder location and apply one of two rules:

- If the declaration names a directory (for example `docs/prps/`), that directory is the root.
  Plans go in `docs/prps/active/<ID>/`.
- If the declaration still ends in `<TICKET-ID>` (the pre-0.4.0 wording), strip that final segment
  and treat the remainder as the root. This keeps existing `CLAUDE.md` declarations working.

Both plugins read the same configured value and resolve it the same way.

## Status vocabulary

Use exactly one of these three status values when stamping a plan's outcome:

- `completed`
- `abandoned`
- `superseded by <TICKET-ID>`

There is no `merged` status. At the point this skill runs, the PR has not merged — that is the
whole point of running it inside the branch so the archive travels in the PR. The status describes
the plan's own outcome; whether the PR merged is GitHub's business, and the PR number is the
pointer to it.

## Header format

Archived plans carry a status comment in the first five lines of `master-plan.md`, and of
`tasks.md` and `handoff.md` when those exist:

```markdown
<!-- STATUS: completed · closed 2026-08-18 · PR #77 -->
# GH-412 — Partial refunds
```

Fields, in this order, separated by ` · `:

| Field | Values | Notes |
|---|---|---|
| status | `completed` \| `abandoned` \| `superseded by <TICKET-ID>` | Describes the **plan**, not the PR |
| closed | `closed YYYY-MM-DD` | Absolute date, never relative |
| PR | `PR #<n>` or `PR —` | Whatever the user supplies or `tasks.md` already holds |

When there is no PR number, write `PR —`.

The header is an HTML comment so it renders as nothing on GitHub while staying greppable, and it
sits above the H1 so `head -5` finds it without parsing.

## The authoritative carrier

`master-plan.md` is the only file you stamp and the only file you read for status. `issue.specs`,
`phases/*.md` and every other file in the folder are not stamped and are not authoritative.

When a search matches any file under `closed/<ID>/`, resolve that plan's status by reading
`closed/<ID>/master-plan.md` — not the file that matched. Do this once per plan folder, regardless
of how many files inside it matched; collapse every match on a folder into the one status you read
from that folder's `master-plan.md`.

If `master-plan.md` is missing, or present but carries no header (a folder moved by hand), report
that plan's status as `unknown`. Never skip it.

## INDEX.md

`INDEX.md` lists closed plans only — active plans are visible with `ls active/`. This keeps
`close-master-plan` the single writer of the index.

```markdown
# Closed plans

| Plan | Title | Status | Closed | PR |
|---|---|---|---|---|
| GH-412 | Partial refunds | completed | 2026-08-18 | #77 |
| GH-388 | Idempotency filter | superseded by GH-412 | 2026-06-02 | #61 |
```

List entries newest first. Create the file on first close if it does not already exist. It is a
convenience for humans; nothing in either plugin's behaviour depends on it being accurate.
