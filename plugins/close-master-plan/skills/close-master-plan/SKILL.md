---
name: close-master-plan
description: This skill should be used to close out a master implementation plan after its work has landed on a branch — reconciling tasks.md against the real commits, verifying handoff.md is complete, stamping the plan's outcome, and archiving the folder under docs/plans/closed/. It never changes git state itself beyond printing the exact commands to run. Trigger when the user invokes `/close-master-plan`, with or without a `<plan-folder>` or `<TICKET-ID>` argument, or asks to "close the plan for GH-412", "archive this plan", "wrap up GH-412".
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
3. Check `<plans-root>/closed/<ID>/` first. If it exists, this plan is already closed: read the
   header from its `master-plan.md` (per `references/plan-layout.md`'s header format), report the
   status and closed date to the user, and **stop** — do not proceed to any later step. If
   `master-plan.md` is missing, or present but carries no header, report status `unknown` and stop
   instead — per `plan-layout.md`'s "authoritative carrier" section, never skip a folder just
   because it can't be read. This is what keeps closing idempotent: a second run against an
   already-closed plan neither re-stamps nor re-moves anything nor duplicates its `INDEX.md` row.
4. Otherwise look for `<plans-root>/active/<ID>/`. If it is not there, fall back to the flat
   `<plans-root>/<ID>/`. **Record which layout was found** — Step 7 needs it to know which `git mv`
   to run.
5. Require `master-plan.md` to exist in whichever folder was found. Without it this is not a plan
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
Step 8 will print a two-part sequence rather than a single commit, and that the close is not
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
phased out cannot be `completed`. Step 6 stamps only the files that exist in the folder, so a
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

### Step 6 — Stamp

Write the header — format in `references/plan-layout.md` — into the plan folder's files:

- **`master-plan.md`**: mandatory. Step 1 already required this file to exist, so this write never
  has a missing target.
- **`tasks.md`** and **`handoff.md`**: stamped **if they exist**. Step 4a's no-`tasks.md` case, and
  a folder where `handoff.md` was never written, both mean fewer files stamped, not a failure.
- **`issue.specs`** and everything under **`phases/*.md`**: never stamped. `plan-layout.md` names
  `master-plan.md` as the only authoritative carrier — stamping the others would create copies of
  the truth that can drift from it.

Assemble the header from what the earlier steps established: the `status` from Step 3, today's date
for `closed`, and the PR field from whatever the user supplied in Step 4 (or `tasks.md`'s existing
`PR` column) or `—`.

If a file already carries a header — this run is re-stamping after a correction, or a stray header
survived a hand-edit — **replace it rather than adding a second.** A file must never end up with two
`<!-- STATUS: ... -->` lines in its first five lines.

### Step 7 — Move and index

The source is whichever layout Step 1 recorded; the destination is always
`<plans-root>/closed/<ID>/`. Ensure `<plans-root>/closed/` exists first (`mkdir -p`) — `git mv`
renames at the filesystem level, so it fails with "No such file or directory" if that parent
directory isn't there yet, which is exactly the case the very first time a project closes a plan:

```sh
git mv docs/plans/active/GH-412 docs/plans/closed/GH-412   # active layout
git mv docs/plans/GH-412        docs/plans/closed/GH-412   # flat legacy layout
```

Use `git mv` because it stages the move in the same step it happens, keeping the index coherent with
the working tree. It does **not** guarantee a rename-rendered diff: whether a diff renders as a
clean rename block is Git's similarity-detection heuristic, not a promise `git mv` makes, and this
close-out edits several of the files inside the folder (the stamped headers, the reconciled
`tasks.md`) in the same commit that moves them. A clean rename block is likely, not promised — don't
tell the user to expect one.

Two situations need different handling:

- **Untracked plan folder** (a plan whose folder was never committed): `git mv` has nothing to stage
  a rename from. Fall back to a plain `mv`, and say so — the move still happens, it just isn't a
  tracked rename.
- **Destination already exists**: stop and list what's there, following the overwrite-aware pattern
  `create-master-plan`'s Step 1 uses for its own folder collision — surface the conflict and let the
  user decide, never silently clobber an existing `closed/<ID>/`. Stop-and-list is the whole
  response here, not `create-master-plan`'s three-option menu: there is no useful "merge" target
  once a plan is already closed, the way there is for a fresh plan draft being re-run.

Once the move is staged, create or update `<plans-root>/INDEX.md` per `references/index-template.md`
— create the file if this is the first close, otherwise insert the new row newest-first.

### Step 8 — Report and stop

By this point two different things have already happened to the plan folder's content, and it's
worth being precise about which: Step 7's `git mv` already staged the move itself, and along with
it whatever was on disk at that moment — the reconciled `tasks.md` and the headers Step 6 stamped,
since both were written before Step 7 ran. What is *not* yet staged is the `INDEX.md` Step 7 just
created or updated (disk-only until now). For `completed` and `superseded`, this step's
`git add` is what stages that — re-adding the already-staged plan-folder paths alongside it
is a harmless no-op. Print:

```sh
git add docs/plans && git commit -m "GH-412: close plan"
```

For `abandoned`, print the same commit **plus** the rescue, both presented as required, not optional
— Step 3 already warned the user this close only becomes durable once the second command runs:

```sh
git add docs/plans && git commit -m "GH-412: close plan (abandoned)"
# the branch is being deleted — the archive must also land on the default branch:
git switch main && git cherry-pick <that-commit>
```

The commit alone is not durable: it sits on a branch about to be deleted, and without the
cherry-pick the archive this skill exists to preserve is lost along with it. Warn the user the
cherry-pick commonly hits rename/delete conflicts — the plan folder usually never existed on the
default branch at all, only on the branch being abandoned, so there is no pre-image for Git to
rename from. Resolve by `git add`-ing the reported paths to accept the incoming files, then
`git cherry-pick --continue`; this is expected, not a sign anything went wrong.

Always stage explicit paths — `docs/plans` — never `git add -A`; the close commit must
contain the close and nothing else. Point the user at `superpowers:finishing-a-development-branch`
for merging, pushing, and worktree removal — that is not this skill's job. Close with the same
statement the Overview opens with: this skill does not commit, push, switch branches, or touch the
worktree it is running inside. The user runs every command it prints.

## Correcting a mistaken close

There is no reopen in v0.1.0. Undoing a close is either `git revert` of the close commit, or a
manual reverse: `git mv` the folder back to `active/`, remove the header from each file it was
stamped into, and delete the row `INDEX.md` gained.

There's no reopen command because a reopen would have to honestly reverse two different kinds of
change at once: restore whichever header the stamp replaced (only recoverable if that prior header
is still visible in git history), and undo the `tasks.md` reconciliation the user built by hand in
Step 4. Automating that
would be claiming a reversibility this skill cannot honestly guarantee — `git revert` or a manual
fix says exactly what happened, instead of pretending the close never occurred.
