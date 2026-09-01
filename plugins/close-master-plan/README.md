# close-master-plan

Step 6 of a plan-first multi-agent workflow: closes out a master implementation plan once its work has landed on a branch. It reconciles `tasks.md` against the real commits, verifies `handoff.md` is complete, stamps the plan's outcome, archives the folder under `docs/plans/closed/`, pushes the branch and opens the PR — and, on a second run once that PR has merged, deletes the branch and removes the worktree.

**It runs those git commands rather than printing them. The one thing it never does is merge.** That is the hinge the design turns on: everything before the merge is reversible, so the skill does it for you; everything after it is safe *because* a human merged, so the merge stays yours. Its contract lives in [`git-lifecycle.md`](skills/close-master-plan/references/git-lifecycle.md), shared byte-identically with [`create-master-plan`](../create-master-plan/), which owns the other end of the same lifecycle.

`MANUAL.html` in this folder documents the complete six-step workflow this plugin closes out, step 6 in particular.

## When to run it

After `/code-review` inside Claude Code and the optional `/plan-implementation-review` from [`plan-review`](../plan-review/) — this is the last step in the pack, not a substitute for either review. Run it once the branch's work is actually done (or actually abandoned).

Then run it **again** after the PR merges. The second run is the cleanup, and it is a separate invocation on purpose: a merge can take minutes or days, it may need CI or a reviewer, and nothing should be deleted until it has actually happened.

---

## Setup

### 1. Install the plugin

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install close-master-plan@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH — pass `https://github.com/necofx/necofx-claude-marketplace.git` instead.

### 2. Nothing else to configure

No required companion plugin, no MCP server, no network call. The skill reads and writes the local repository with ordinary tools and `AskUserQuestion`.

If your plans live somewhere other than `docs/plans/`, that root is one setting shared by all four plugins in the pack — see [`create-master-plan`'s config.md](../create-master-plan/config.md#a-different-plans-root). `close-master-plan` resolves the same `CLAUDE.md` declaration `create-master-plan` does, the same way.

### 3. Restart

Skills load at session start. Open a new conversation before your first `/close-master-plan`.

---

## Usage

```
/close-master-plan [<plan-folder> | <TICKET-ID>]
```

Give it a path, a ticket id like `GH-412`, or nothing — with no argument it looks in `<plans-root>/active/` first, proposes the one folder it finds there (or asks which, if there are several), and falls back to a flat legacy `<plans-root>/<ID>/` folder under the same rule.

The same command runs one of two phases, chosen from where the plan folder sits and what `gh` says about its PR — never from a stored flag:

| Phase | When | What it does |
|---|---|---|
| **1 · propose** | plan is in `active/` | reconcile → verify → push → open PR → stamp → archive → commit → push |
| **2 · clean** | plan is in `closed/` and its PR is `MERGED` | back to the default branch → pull → delete branch (local + remote) → remove worktree |

A run that finds the plan closed but its PR still open reports that and stops — there is nothing to clean until someone merges.

Phase 1, in order:

1. **Locate and validate.** Checks `closed/<ID>/` first — if the plan is already there, this is the phase-2 entry point: it asks `gh` whether the PR merged, and either cleans up or explains why it can't yet. Otherwise finds the plan under `active/<ID>/` or the flat legacy layout, and requires `master-plan.md` to exist.
2. **Git preflight.** The working tree must be clean — a close commit that also contains unfinished work defeats the point of the archive. Resolves the base branch and the branch's commit list, and warns (without blocking) if the run looks like it's happening from the wrong checkout, or if no implementation-review artifact is present in the folder.
3. **Establish the status**, via `AskUserQuestion`: `completed`, `abandoned`, or `superseded by <TICKET-ID>`. There is no `merged` status — at the point this runs, the PR hasn't merged yet, and the PR number is the durable pointer to that outcome.
4. **Reconcile `tasks.md`.** Proposes a phase-to-commit mapping built from `tasks.md`'s Detailed Progress entries and the branch's commit subjects, and has you correct it before applying it — commit messages carry no contracted format, so the mapping is never derived silently. Fills in `Finished` timestamps and a `Final Summary`. A plan can't be stamped `completed` while any phase is still `pending`, `in_progress`, or `blocked`; mark it `dropped` with a justification, or finish the phase first. (Skipped entirely if the folder has no `tasks.md` — a plan that was never decomposed can only close as `abandoned` or `superseded`.)
5. **Verify `handoff.md`.** Scans it for template placeholders left behind, and asks you to confirm out loud that an empty "Key deviations from the original plan" section really is empty — that's the section reviewers read most closely.
5.5. **Push and open the PR.** Pushes the branch, then opens a PR titled from the `master-plan.md` H1 with a body assembled from `handoff.md` — what shipped, deviations, how to verify, review focus — plus `Closes #<n>`, ready for review. **This happens before the stamp on purpose:** the header records `PR #<n>`, and that number doesn't exist until the PR does; opening it first avoids a second "record the number" commit or an amend plus force-push. An existing PR for the branch is reused, never duplicated. No `gh`? It pushes and prints the command for you instead of failing.
6. **Stamp.** Writes a one-line status header into `master-plan.md` (and into `tasks.md` / `handoff.md`, if they exist) recording the status, the close date, and the PR number from the previous step. That number is what phase 2 reads back.
7. **Move and index.** `git mv`s the folder to `<plans-root>/closed/<ID>/` and updates `<plans-root>/INDEX.md` with a newest-first row. Stops and lists what's there rather than overwriting if the destination already exists.
8. **Commit, push, hand back.** Runs the close commit and pushes it into the PR opened in 5.5, then tells you plainly what's true: archived, pushed, PR open — **and the merge is yours**. Run the command again after it merges. For `abandoned`, it runs a second command too — see below.

---

## Three things to understand before you run it

### Why the merge isn't automated

Everything phase 1 does is reversible: a commit can be reverted, a PR can be closed, a pushed branch can be deleted. Everything phase 2 does — deleting the branch, removing the worktree — is only safe *because* the content already reached the default branch. The merge is the line between those two worlds, and it's the one action whose correctness the skill cannot check for you: whether CI passed, whether a reviewer agrees, whether this should ship at all. So it stops there, hands you the PR, and waits to be run again.

That also means phase 2 verifies the merge with `gh` rather than trusting the header. A header that names a PR is not evidence the PR merged.

### `abandoned` has a two-part ending

If you close a plan as `abandoned`, the skill still performs the close **on the current branch**, because the plan folder's content only exists there right now. But a commit on a branch that's about to be deleted is not durable — deleting the branch takes the archive with it, which is exactly the record this skill exists to preserve.

So for `abandoned`, Step 8 runs two commands, both required, not one optional follow-up — and this is the **only** path in the skill that confirms each command with you before running it, because its expected outcome is that a branch carrying real work disappears:

```sh
git add docs/plans && git commit -m "GH-412: close plan (abandoned)"
# the branch is being deleted — the archive must also land on the default branch:
git switch main && git cherry-pick <that-commit>
```

Expect the cherry-pick to hit rename/delete conflicts — the plan folder usually never existed on the default branch, only on the branch being abandoned, so there's no pre-image for Git to rename from. Resolve by `git add`-ing the reported paths and running `git cherry-pick --continue`; that's the normal path, not a sign something went wrong. `abandoned` opens no PR, so there's no phase 2 for it.

### There is no reopen

Closing the wrong plan, or closing with the wrong status, has no dedicated undo command. Use `git revert` on the close commit, or reverse it by hand: `git mv` the folder back to `active/`, strip the header from each file it was stamped into, and delete the row `INDEX.md` gained.

There's no automated reopen because it would have to honestly reverse two different kinds of change at once: a header that replaced whatever was there before (only recoverable from git history), and a `tasks.md` reconciliation you built by hand. Automating that would claim a reversibility the skill can't actually guarantee — `git revert` or a manual fix says exactly what happened, instead of pretending the close never occurred.

---

## The `active/` / `closed/` plan-folder layout

`close-master-plan` (together with `create-master-plan` 0.4.0) introduces this layout under `<plans-root>`, which defaults to `docs/plans/`:

```
docs/plans/
  INDEX.md            # closed plans only, one line each, newest first
  active/
    GH-500/           # a plan being worked
  closed/
    GH-412/           # completed
    GH-388/           # superseded by GH-412
```

`<plans-root>` is a root, not a per-ticket pattern, and it's the same setting across all four plugins in the pack — `create-master-plan`, `decompose-plan`, `plan-review`, and this one all have to agree on it, or one plugin can't find what another wrote. See [`create-master-plan`'s config.md](../create-master-plan/config.md#a-different-plans-root) to point it somewhere other than `docs/plans/`.

The flat legacy layout — a plan folder directly under `<plans-root>/<TICKET-ID>/`, no `active`/`closed` split — still works: this skill finds it and closes it in place, and migrating existing plans onto the new layout is left to you, not automated.

`master-plan.md` is the only authoritative carrier inside a plan folder. Its status header is what this skill reads on a re-run, and the only file it's guaranteed to stamp — `issue.specs` and everything under `phases/` are never stamped, so they can never drift out of sync with a copy of the truth.

---

## When it goes wrong

| Symptom | Cause and fix |
|---|---|
| "Working tree not clean" | The close commit must contain only the close. Commit or stash unrelated changes first. |
| Won't stamp `completed` | A phase is still `pending`, `in_progress`, or `blocked`. Finish it, or mark it `dropped` with a justification under `tasks.md`'s `Decisions`. |
| "Already closed" and it stops | `<plans-root>/closed/<ID>/` exists and its PR hasn't merged yet. Merge it, then run the command again — that run is the cleanup. If the header carries no PR at all, there's nothing for phase 2 to verify and it stops for good. |
| Phase 2 says the worktree is already gone | Expected, and not an error. The harness offers to remove the worktree when a session ends, so by the time you merge it usually has. Every cleanup step checks first and reports what was already done. |
| Phase 2 refuses: unpushed work in the worktree | A merged PR says nothing about what someone added afterwards. Push it or discard it, then re-run — this guard is the one thing standing between a stray commit and a deleted branch. |
| `git branch -d` says "not fully merged" | A squash merge puts the branch's content on the default branch under a different SHA. The skill uses `-D` here, but only after `gh` confirmed `MERGED`, and it says so when it does. |
| Destination already exists at Step 7 | Something is already at `closed/<ID>/`. The skill lists what's there and stops; there's no merge option for an already-closed folder, only your judgment call. |
| Base branch is ambiguous | Asked via `AskUserQuestion` rather than guessed — diffing a committed branch against `HEAD` would vacuously find nothing to close. |
| Warned about running from the wrong checkout | The branch name doesn't reference the ticket id and the plan folder has no history on this branch — the pack's most common mistake is running from the main checkout while the real work is in a worktree. Informational, not a gate. |
| Plan folder was never committed | `git mv` has nothing to rename from. The skill falls back to a plain `mv` and says so — the move still happens, it just isn't a tracked rename. |

## Limits

- **It never merges.** Commit, push, PR, branch deletion and worktree removal are all its job; the merge is yours, and phase 2 won't run until `gh` confirms it happened.
- **It never force-pushes and never rewrites history.** The PR-before-stamp ordering exists precisely so no amend is needed.
- **No reopen command.** See [above](#there-is-no-reopen).
- **Cleanup needs a PR to verify against.** A plan stamped `PR —` (no `gh`, no remote, or a close done by hand) has nothing phase 2 can check, so the branch and worktree stay for you to remove.
- **A plan with no `tasks.md` can only close as `abandoned` or `superseded`.** It was never decomposed into phases, so there's nothing to reconcile against commits or verify in `handoff.md`.
- **Migrating an existing flat-layout project onto `active/`/`closed/` is manual.** Both this skill and `create-master-plan` keep working with the flat layout indefinitely; nothing forces the move.
