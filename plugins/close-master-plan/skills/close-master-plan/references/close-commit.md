# Step 8 — Commit, push, and hand back

_Moved out of `SKILL.md` so it survives context compaction: the skill reads this file from disk at the step that needs it._

### Step 8 — Commit, push, and hand back

By this point two different things have already happened to the plan folder's content, and it's
worth being precise about which: Step 7's `git mv` already staged the move itself, and along with
it whatever was on disk at that moment — the reconciled `tasks.md` and the headers Step 6 stamped,
since both were written before Step 7 ran. What is *not* yet staged is the `INDEX.md` Step 7 just
created or updated (disk-only until now). This step's `git add` is what stages that — re-adding the
already-staged plan-folder paths alongside it is a harmless no-op.

For `completed` and `superseded`, **run** this, then push:

```sh
git add docs/plans && git commit -m "GH-412: close plan"
git push
```

The close commit lands in the PR Step 5.5 opened, seconds after it. The reviewer sees the work and
its close-out in one place.

For `abandoned`, the same commit **plus** the rescue — and this is the one path in the skill that
**confirms each command with the user before running it**, because its expected outcome is that a
branch carrying real work disappears:

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

Always stage explicit paths — `docs/plans` — never `git add -A`; the close commit must contain the
close and nothing else.

Then **report and stop**, stating plainly what is now true and what is not: the plan is archived,
the branch is pushed, the PR is open at `<url>` — **and the merge is yours**. Tell the user to run
`/close-master-plan <ID>` again once it has merged, and that the second run is what deletes the
branch and removes the worktree. Nothing else in this skill runs until then.
