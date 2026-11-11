# Phase 2 — clean up, and correcting a mistaken close

_Moved out of `SKILL.md` so it survives context compaction: the skill reads this file from disk at the step that needs it._

### Step 9 — Clean up  *(phase 2, reached only from Step 1)*

Step 1 sends the run here when the plan is already in `closed/` and `gh` confirmed its PR is
`MERGED`. Apply `references/git-lifecycle.md`'s phase-2 section. Steps 2 through 8 do not run: there
is nothing left to reconcile, verify, stamp or move.

1. **Guard first.** If the worktree holds uncommitted changes or unpushed commits, stop and list
   them. Cleanup would destroy work that exists nowhere else, and a merged PR says nothing about
   what someone added afterwards.
2. **Leave and remove in one operation:** `ExitWorktree` with `action: "remove"`, which returns the
   session to the directory it started in. A worktree cannot be removed from inside itself — this is
   the harness's own answer to that, and the reason the skill no longer needs to hand the problem
   back to the user. When the session is *not* in a worktree (phase 2 run later from the main
   checkout), use `git worktree remove <path>` instead, which is safe from there.
3. `git switch <default-branch> && git pull`. The merge already brought `closed/<ID>/` and the
   updated `INDEX.md` to the default branch, so this is where the archive becomes visible outside
   the branch.
4. **Delete the branch, local and remote.** After a squash merge `git branch -d` fails with "not
   fully merged", because the branch's content reached the default branch under a different SHA.
   Use `-D` **only** because Step 1 already confirmed `MERGED`, and say that out loud when you do —
   an unexplained `-D` reads like data loss. Then `git push origin --delete <branch>`, tolerating a
   remote branch that is already gone because the repository auto-deletes on merge.
5. **Report** what was cleaned, what was already gone, and where the session is standing now.

**Every step here is idempotent and optional.** Between the two phases a human closes sessions — and
the harness asks whether to keep or remove the worktree on exit — so by the time phase 2 runs the
usual case is that the worktree is *already* gone, the branch may be gone too, and the session may
already be on the default branch. Check, act if there is something to do, and report what was
already done. **Phase 2 never fails because something already happened.**

If CodeGraph left a daemon holding files under the worktree, stop it before removing the tree; if
removal still fails, report it and let the user re-run rather than forcing it.

## Correcting a mistaken close

There is no reopen command. Undoing a phase-1 close is `git revert` of the close commit, or a manual
reverse: `git mv` the folder back to `active/`, remove the header from each file it was stamped
into, and delete the row `INDEX.md` gained. Close the PR Step 5.5 opened, or leave it open if the
work is still going somewhere.

There's no reopen command because a reopen would have to honestly reverse two different kinds of
change at once: restore whichever header the stamp replaced (only recoverable if that prior header
is still visible in git history), and undo the `tasks.md` reconciliation the user built by hand in
Step 4. Automating that would be claiming a reversibility this skill cannot honestly guarantee —
`git revert` or a manual fix says exactly what happened, instead of pretending the close never
occurred.

**Phase 2 is the safer half, by construction.** It only ever runs after `gh` confirmed the PR
merged, so everything it deletes is redundant with the default branch: the branch's content is on
`main` under some SHA, and the worktree was a second checkout of a branch that no longer has unique
work in it. If a deleted branch is wanted back, `git switch -c <branch> <merge-commit>` recreates
it; if the worktree is wanted back, make another one. The one thing phase 2 refuses to do is run
before that confirmation — which is exactly why the merge is not this skill's to perform.
