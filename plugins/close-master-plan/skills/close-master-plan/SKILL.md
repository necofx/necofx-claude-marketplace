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
- If it is empty, fall back to the flat layout (point 3 above).
- If that finds nothing either, stop and say so — there is no plan folder to close.

### Step 2 — Git preflight

Establish, in this order:

1. **Is this a git repository.** Every later step depends on it — if not, stop.
2. **Is the working tree clean.** **If it is not, stop.** State the reason: the close commit must
   contain the close and nothing else, and mixing it with unfinished work destroys the property
   that makes every later step safe to inspect.
3. **Whether this checkout is a worktree**, via `git rev-parse --git-dir` versus
   `git rev-parse --git-common-dir` — they differ in a worktree, match in a normal checkout — and
   which branch is checked out.
4. **The base branch**, via `git merge-base` against the repository's default branch. If it is
   ambiguous, ask the user via `AskUserQuestion` — once work is committed on a branch, the diff
   base is no longer `HEAD`; asking against `HEAD` on a committed branch would find a clean tree
   and vacuously report nothing to close.
5. **The branch's commits**: `git log --oneline --no-merges <base>..HEAD`. This is the raw material
   Step 4 reconciles against `tasks.md`.

Then two warnings. Neither of these blocks the run — they inform the user, they do not gate the
close:

- **Worktree/main-checkout mismatch.** If the branch name does not reference the ticket id *and*
  the plan folder shows no history on this branch, warn that the run may be happening from the
  main checkout while the actual work lives in a worktree — the pack's most common mistake. Every
  path the rest of this skill touches is relative to wherever it is actually running.
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
