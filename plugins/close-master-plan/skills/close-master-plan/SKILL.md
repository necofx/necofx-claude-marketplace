---
name: close-master-plan
description: This skill should be used to close out a master implementation plan after its work has landed on a branch — reconciling tasks.md against the real commits, verifying handoff.md is complete, stamping the plan's outcome, archiving the folder under docs/plans/closed/, then pushing the branch and opening the PR; a second run, once that PR has merged, deletes the branch and removes the worktree. It performs these git operations rather than printing them, but never merges — that stays a human decision. Trigger when the user invokes `/close-master-plan`, with or without a `<plan-folder>` or `<TICKET-ID>` argument, or asks to "close the plan for GH-412", "archive this plan", "wrap up GH-412", "clean up after GH-412".
---

# Close Master Plan

## Overview

Closes out a master implementation plan once its work is done. This runs **after** `/code-review`
and `/plan-implementation-review` — it is the last step in the pack, not a substitute for either
review.

**It owns the git side of the close, and runs the commands rather than printing them** — the commit,
the push, the PR, and later the branch and worktree deletion. The single thing it never does is
**merge**. That is the hinge the whole design turns on: everything before the merge is reversible
and the skill does it for you; everything after it is only safe *because* a human merged, so the
merge stays yours. The full contract is `references/git-lifecycle.md`, shared byte-identically with
`create-master-plan`.

That splits this skill into two phases of the same command:

- **Phase 1 — propose.** The plan folder is still in `active/`: reconcile, verify, push, open the
  PR, stamp, archive, commit, push.
- **Phase 2 — clean.** The plan folder is already in `closed/` and its header's PR has merged: back
  to the default branch, pull, delete the branch local and remote, remove the worktree.

Which phase runs is **derived, never stored** — from where the plan folder sits and what `gh` says
about the PR the header already records. Running the command a third time reports that there is
nothing left to do.

## Inputs

- **`<plan-folder>` or `<TICKET-ID>`** (optional): a path to a plan folder, or a ticket id like
  `GH-412`. Step 1 resolves whichever form is given, or locates the folder itself when the
  argument is absent.

## Workflow

Read `references/git-lifecycle.md` first — it defines the states, the ordering constraints and the
degradation rules every step below depends on. Then follow these steps in order.

### Step 1 — Locate and validate

1. Resolve the project root (nearest ancestor of CWD with a `.git` directory).
2. Resolve `<plans-root>` per `references/plan-layout.md`.
3. Check `<plans-root>/closed/<ID>/` first. If it exists, this plan is already closed — **this is
   the phase-2 entry point.** Read the header from its `master-plan.md` (per
   `references/plan-layout.md`'s header format) and branch on what it says:
   - **Header names a PR** → ask `gh` whether it merged (`gh pr view <n> --json state,mergedAt`).
     `MERGED` means go to **Step 9 — Clean up**, skipping Steps 2 through 8 entirely: there is
     nothing left to reconcile, verify, stamp or move. Anything else (still open, closed unmerged)
     means report that state and **stop** — there is nothing to clean until a human merges.
   - **Header names no PR** → report the status and closed date and stop, as before.
   - **`master-plan.md` is missing, or present with no header** → report status `unknown` and stop —
     per `plan-layout.md`'s "authoritative carrier" section, never skip a folder just because it
     can't be read.

   This is what keeps closing idempotent. A second run neither re-stamps nor re-moves anything nor
   duplicates its `INDEX.md` row; it either does the cleanup the merge made safe, or explains why it
   is not safe yet.
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
  checkout while the actual work lives in the worktree `create-master-plan` made — the pack's most
  common mistake, and the reason Step 0.5 of that skill reports the worktree path it left you in.
  Every path the rest of this skill touches is relative to wherever it is actually running.
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
Step 8 will run a two-part sequence rather than a single commit — **confirming each command first,
the only place in this skill that does** — and that the close is not durable until that second part
runs. `abandoned` also skips Step 5.5 entirely: a branch that is about to be deleted gets no PR, so
there is no merge for phase 2 to wait on and no phase 2 to run.

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

### Step 5.5 — Push the branch and open the PR

Skip this step for `abandoned` — a branch about to be deleted gets no PR; Step 8's rescue path
handles that case instead.

**This runs before the stamp, and the ordering is the whole point.** The header records `PR #<n>`,
and that number does not exist until the PR does. Opening it first is what avoids a second "record
the number" commit or an `--amend` plus force-push over an already-pushed branch:

```sh
git push -u origin <branch>
gh pr create --title "<master-plan.md's H1>" --body "<from handoff.md>" --base <default-branch>
```

- **If a PR for this branch already exists**, reuse its number and update its body — never open a
  second one for the same branch.
- **Title** is the `master-plan.md` H1. **Body** is assembled from `handoff.md`: what shipped, the
  deviations, how to verify, review focus — plus a link to the ticket and a `Closes #<n>` line when
  the source was a GitHub issue. Step 5 verified that file one step earlier, so this is material
  that already exists rather than prose invented at PR time.
- **Opened ready for review, not draft.**
- **If `gh` is missing or unauthenticated**, push anyway and print the `gh pr create` command for
  the user to run, then continue the close with `PR —` in the header. Degrade, never fail.
- **If there is no remote at all**, say so, skip the push and the PR, and continue with `PR —`.

Hand the resulting number to Step 6.

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
for `closed`, and the PR field from the number Step 5.5 just created — falling back to whatever the
user supplied in Step 4 (or `tasks.md`'s existing `PR` column), or `—` when Step 5.5 was skipped or
degraded. **That number is what phase 2 reads back**, so a header stamped `PR —` means the cleanup
has nothing to verify a merge against and will stop rather than guess.

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

### Step 8 — Commit, push, and hand back

**Read `references/close-commit.md` now and follow it.** In short: stage `docs/plans` explicitly (never `git add -A`), commit `<ID>: close plan`, push into the PR Step 5.5 opened. For `abandoned` the commit is followed by a cherry-pick onto the default branch, and that path confirms each command with the user first. Then report and stop — **the merge is the user's**, and a second `/close-master-plan <ID>` after it merges is what cleans up.

### Step 9 — Clean up  *(phase 2, reached only from Step 1)*

Step 1 sends the run here when the plan is already in `closed/` and `gh` confirmed its PR is `MERGED`. Steps 2 through 8 do not run. **Read `references/cleanup.md` now and follow it** — the guard against unpushed work, leaving and removing the worktree, switching and pulling the default branch, deleting the branch local and remote, and the report. Every step there is idempotent: phase 2 never fails because something already happened.

## Correcting a mistaken close

There is no reopen command: undo a phase-1 close with `git revert` of the close commit. The full reasoning, and the manual reverse, are in `references/cleanup.md` § Correcting a mistaken close.
