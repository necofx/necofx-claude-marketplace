---
name: close-master-plan
description: This skill should be used to close out a master implementation plan after its work has landed on a branch — reconciling tasks.md against the real commits, verifying handoff.md is complete, distilling durable lessons into .claude/rules/, stamping the plan's outcome, and archiving the folder under docs/plans/closed/. It never changes git state itself beyond printing the exact commands to run. Trigger when the user invokes `/close-master-plan`, with or without a `<plan-folder>` or `<TICKET-ID>` argument, or asks to "close the plan for GH-412", "archive this plan", "wrap up GH-412".
---

# Close Master Plan

## Overview

Closes out a master implementation plan once its work is done. This runs **after** `/code-review`
and `/plan-implementation-review` — it is the last step in the pack, not a substitute for either
review. It never changes git state: every step that touches files stages them, and the skill stops
short of committing, pushing, switching branches, or touching the worktree it is running inside.
The user runs the commands it prints.

## Inputs

- **`<plan-folder>` or `<TICKET-ID>`** (optional): a path to a plan folder, or a ticket id like
  `GH-412`. Step 1 resolves whichever form is given, or locates the folder itself when the
  argument is absent.

## Workflow

Follow these steps in order.

### Step 1 — Locate and validate

1. Resolve the project root (nearest ancestor of CWD with a `.git` directory).
2. Resolve `<plans-root>` per `references/plan-layout.md`.
3. Look for `<plans-root>/active/<ID>/`. If it is not there, fall back to the flat
   `<plans-root>/<ID>/`. **Record which layout was found** — Step 8 needs it to know which `git mv`
   to run.
4. Require `master-plan.md` to exist in whichever folder was found. Without it this is not a plan
   folder — stop and say so.

**No-argument behaviour.** When the user gave neither a plan folder nor a ticket id:

- If `<plans-root>/active/` holds exactly one folder, propose it and confirm with the user before
  proceeding.
- If it holds several, ask which one via `AskUserQuestion`.
- If `<plans-root>/active/` is empty or absent, look for flat-legacy plan folders instead: list
  `<plans-root>/*`, excluding `active/`, `closed/` and `INDEX.md`. Apply the same rule — exactly
  one candidate: propose and confirm; several: ask which one via `AskUserQuestion`; none: stop and
  say so, there is no plan folder to close.

### Step 2 — Git preflight

Establish, in this order:

1. **Is this a git repository.** Every later step depends on it — if not, stop.
2. **Is the working tree clean.** **If it is not, stop.** State the reason: the close commit must
   contain the close and nothing else, and mixing it with unfinished work destroys the property
   that makes every later step safe to inspect.
3. **Whether this checkout is a worktree**, via `git rev-parse --git-dir` versus
   `git rev-parse --git-common-dir` — they differ in a worktree, match in a normal checkout — and
   which branch is checked out.
4. **The base branch**, via `git merge-base` against the repository's default branch — the branch
   `origin/HEAD` points at, or `main`/`master` if there is no `origin`. If it is ambiguous, ask the
   user via `AskUserQuestion` — once work is committed on a branch, the diff base is no longer
   `HEAD`; asking against `HEAD` on a committed branch would find a clean tree and vacuously report
   nothing to close.
5. **The branch's commits**: `git log --oneline --no-merges <base>..HEAD`. This is the raw material
   Step 4 reconciles against `tasks.md`.

Then two warnings. Neither of these blocks the run — they inform the user, they do not gate the
close:

- **Worktree/main-checkout mismatch.** If the branch name does not reference the ticket id *and*
  the plan folder shows no history on this branch (`git log --follow -- <plan-folder>` returns
  nothing older than this branch's own commits), warn that the run may be happening from the main
  checkout while the actual work lives in a worktree — the pack's most common mistake. Every path
  the rest of this skill touches is relative to wherever it is actually running.
- **Absent implementation review.** If no implementation-review artifact
  (`codex-implementation-review.md` or equivalent) is present in the plan folder, say so: closing
  now stamps and archives a folder that a later review may send the user back into. That review is
  optional, so this is information, not a gate — continue regardless.

### Step 3 — Establish the status

Ask the user via `AskUserQuestion`, offering exactly these three values, verbatim, and no others
(there is no `merged` status — see `references/plan-layout.md`):

- `completed`
- `abandoned`
- `superseded by <TICKET-ID>` — when chosen, prompt for the superseding ticket id and substitute it
  into the value.

**If the answer is `abandoned`**, tell the user what that means before continuing: the branch is
about to be deleted unmerged, so a close commit made on it is discarded along with it, and
`closed/<ID>/` never reaches the default branch — losing exactly the record this skill exists to
preserve. The close still happens here, on this branch, because the plan folder's content only
lives on this branch right now — there is nowhere else to make the change. But make clear that
Step 9 will print a two-part sequence rather than a single commit, and that the close is not
durable until that second part runs.

### Step 4 — Reconcile `tasks.md`

If the plan folder has no `tasks.md`, skip this step and Step 5 — go to Step 4a instead.

1. Read `tasks.md`'s phase table and Detailed Progress entries alongside the commit list Step 2
   produced.
2. Build the best phase-to-commit mapping you can from the Detailed Progress entries and the commit
   subjects, **present it as a table, and have the user correct it.** Do not derive it silently:
   `tasks-template.md` requires the coordinator to replace `(pending batch)` with SHAs and to note
   which phases a batch covered, but it imposes **no commit-message format**, and the tutorial
   presents one-commit-per-round as what the coordinator *proposes*, not a contract. Rows the user
   cannot place stay `(pending batch)` rather than being guessed.
3. Once the user confirms (or corrects) the mapping, apply it: replace `(pending batch)` in each
   confirmed row's `Commits` column with the short SHA(s), comma-separated.
4. Fill `Finished` with a timestamp on any `completed` row that is missing one.
5. Fill the `PR` column if the user has a PR number or URL to supply; leave `—` otherwise.
6. Write `Final Summary`, ending with the standing caveat: **the SHAs are feature-branch commits;
   if this PR is squash-merged they will not exist on the default branch, and the PR number is the
   durable pointer.**
7. Update `**Last updated:**` to today's date.

Then stop and ask. If any phase is still `pending`, `in_progress`, or `blocked`, the plan cannot be
stamped `completed` as-is: either mark that phase `dropped` and add a justification line under
`Decisions` — `tasks-template.md`'s own mechanism for recording this — or stop the close here so
the user can go finish the phase first.

### Step 4a — No `tasks.md`

If the plan folder has no `tasks.md`, the plan was never decomposed into phases. Skip Step 4 and
Step 5 entirely — there is nothing to reconcile against commits and nothing to verify. Restrict the
status established in Step 3 to `abandoned` or `superseded by <TICKET-ID>`: a plan that was never
phased out cannot be `completed`. Step 7 stamps only the files that exist in the folder, so a
plan folder with only `master-plan.md` is stamped there and nowhere else.

### Step 5 — Verify `handoff.md`

Skip this step if Step 4a applied.

1. Scan `handoff.md` for everything listed in `references/closeout-checklist.md`.
2. Report every hit found — quote the surrounding line so the user can see it in context — and
   offer to fill them together before continuing.
3. Check the "Key deviations from the original plan" section. An empty section is not an automatic
   failure — a plan can be executed exactly as written — but confirm out loud with the user that it
   is genuinely empty rather than simply unfilled, because it is the section reviewers spend the
   most attention on.

### Step 6 — Distil durable lessons

Read `handoff.md`'s deviations and `tasks.md`'s `Decisions` and `Coordination Notes`, apply
`references/rule-distillation.md`'s durability test to each, and present every candidate that
passes as a concrete diff against the matching `.claude/rules/*.md` file. Approve candidates one at
a time — never write anything without an explicit yes on that specific candidate. Zero approved
candidates is a valid, common outcome; say so rather than manufacturing a rule. Record the list of
files actually written (after approval) — Step 7 needs it for the header's `rules:` field.
