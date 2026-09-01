# Plan git lifecycle

This is the contract `create-master-plan` and `close-master-plan` share for the git side of a
ticket: where the work happens, when an index is built, how the branch reaches a PR, and how
everything is cleaned up afterwards. Both plugins carry a byte-identical copy of this file — edit
one, copy it to the other.

**This pack owns the git lifecycle.** That is a deliberate reversal of an earlier boundary, where
worktrees came only from `superpowers` or the harness and `close-master-plan` printed commands
instead of running them. It now creates the worktree, commits, pushes, opens the PR and removes the
worktree. What it still never does is **merge** — that stays a human decision, and it is the hinge
the whole two-phase close turns on.

## The four states

| State | Reached by | What is true |
|---|---|---|
| `unforked` | the starting point | the main checkout, no ticket branch |
| `forked` | `create-master-plan` | a worktree under `.claude/worktrees/<ID>`, a ticket branch, a local CodeGraph index when the parent repo had one |
| `proposed` | `close-master-plan` phase 1 | plan reconciled, stamped, archived to `closed/`, committed, pushed, PR open |
| `clean` | `close-master-plan` phase 2 | back in the main checkout, pulled, branch deleted local + remote, worktree removed |

## State is derived, never stored

Never write a state file. Every transition is decided from what git and `gh` already know:

- **Am I in a worktree?** `git rev-parse --git-dir` differs from `git rev-parse --git-common-dir`
  in a worktree, and matches in a normal checkout.
- **Which phase of the close is this?** Where the plan folder sits: `active/` → phase 1;
  `closed/` → the plan is closed, so read its `master-plan.md` header.
- **Did the PR merge?** `gh pr view <n> --json state,mergedAt`. The header's `PR #<n>` field —
  already the only thing this pack persists about a closed plan — is what carries the number from
  phase 1 to phase 2.

A stored state is a second source of truth that drifts from git the first time a human does
anything by hand. Between phase 1 and phase 2 a human usually *does* do something by hand, so
deriving is not a purity argument — it is the only thing that survives the gap.

## Creation side

### When to fork

**Between the argument parse and the folder setup — not after.** The plan folder is created in
whatever tree the session is standing in; fork later and the folder was written into the main
checkout while the worktree is a clean checkout of the branch that does not contain it. Forking
first means `issue.specs`, the interview and `master-plan.md` are all born inside the worktree.

The cost of that ordering is that the branch name can only use the ticket id — the ticket's title
is not fetched yet. That is the right trade: a folder in the wrong tree is a real bug, an
id-only branch name is cosmetic.

### Naming

`EnterWorktree({name: "<ticket-id-lowercased>"})` creates the worktree at
`.claude/worktrees/<id>` on a branch the **harness** names `worktree-<id>` — the prefix is not
ours and cannot be passed in. Rename it immediately, before anything is pushed:

```sh
git branch -m feature/<id>
```

The base ref comes from the `worktree.baseRef` setting, whose default (`fresh`) branches from
`origin/<default-branch>` rather than local HEAD — the right base for a ticket.

If `.claude/worktrees/<id>` already exists from an earlier run, **enter it** with
`EnterWorktree({path: ...})` rather than creating a second one. This mirrors the overwrite-aware
handling the plan folder already gets.

### CodeGraph

Build a worktree-local index **only when `<parent-repo>/.codegraph/` already exists**. A parent
index is the user's standing decision to use CodeGraph in that repository, and the worktree
inherits it; a repository with no index is a repository whose owner never opted in, and nothing
here changes that. This is the single exception to the pack's "never index a repository yourself"
rule, and it exists because the index is disposable: it lives inside the worktree, self-gitignores,
and disappears with the worktree in phase 2.

Without a local index, CodeGraph still answers from the parent's — it resolves upward and warns
that results reflect a different working tree, so symbols created only in the worktree are missing.
At plan time that is harmless, because the worktree is still identical to its base. The drift
appears during execution, which is exactly what the local index is for.

Start it in the background immediately after entering the worktree:

```sh
codegraph init
```

Then continue with the ticket fetch, the attachments and the `docs/` scan — none of them need the
index. The first consumer is the interview; check `codegraph status` before relying on it there and
fall back to ordinary search if it has not finished. Report the resulting index size once it has,
rather than asking permission for it beforehand.

### Escapes — applied silently, never as a question

| Situation | Behaviour |
|---|---|
| Already inside a worktree | Stay in it and say so — `EnterWorktree` refuses to nest |
| Not a git repository | Skip the fork and the index, continue normally |
| No `origin` remote | `baseRef: fresh` cannot resolve `origin/HEAD`; falls back to local HEAD, and says so |
| The harness cannot move the session's cwd | Continue unforked, and say so |
| Parent repo has no `.codegraph/` | Build no index — the standing rule is untouched |

The run summary must state which worktree and branch the session ended up in, and whether it is
reading a local index or the parent's. Without those three lines the conversation ends without the
user knowing where they are standing.

## Close side, phase 1 — propose

Phase 1 applies when the plan folder is still in `active/` (or the flat legacy layout).

**The PR is opened before the header is stamped.** The header records `PR #<n>`, and that number
does not exist until the PR does. Opening it first is what avoids a second "record the number"
commit or an `--amend` plus force-push:

1. Locate, git preflight, establish the status, reconcile `tasks.md`, verify `handoff.md` — as
   before.
2. `git push -u origin feature/<id>`, then `gh pr create` against the work commits already on the
   branch. If a PR for this branch already exists, **reuse its number** and update its body rather
   than opening a second.
3. Stamp the header — now with the real `PR #<n>` — move the folder to `closed/`, update `INDEX.md`.
4. `git add docs/plans && git commit`, then `git push`. The close commit lands in the same PR
   seconds later.

One extra push, no rewritten history, and the reviewer sees the work and its close-out together.

**PR content.** Title: the `master-plan.md` H1. Body: assembled from `handoff.md` — what shipped,
deviations from the plan, how to verify, review focus — plus a link to the ticket and a
`Closes #<n>` line when the source was a GitHub issue. Opened ready for review, not draft.
`handoff.md` was verified complete one step earlier, so this is material that already exists rather
than prose invented at PR time.

## Close side, phase 2 — clean

Phase 2 applies when the plan folder is already in `closed/` and its header names a PR.

1. **Confirm the merge for real** with `gh pr view <n> --json state,mergedAt`. A header that names
   a PR is not evidence that the PR merged.
2. **Guard:** if the worktree holds uncommitted changes or unpushed commits, stop and list them.
   Cleanup would destroy work that exists nowhere else.
3. **Leave and remove in one operation:** `ExitWorktree` with `action: "remove"`, which returns the
   session to its original directory. A worktree cannot be removed from inside itself, and this is
   the harness's own answer to that. When the session is *not* in a worktree — phase 2 run later
   from the main checkout — use `git worktree remove <path>` instead, which is safe from there.
4. `git switch <default-branch> && git pull`. The merge already brought `closed/<ID>/` and
   `INDEX.md` to the default branch.
5. Delete the branch, local and remote. After a **squash** merge `git branch -d` fails with "not
   fully merged", because the branch's content reached the default branch under a different SHA.
   Use `-D` **only** once step 1 confirmed `MERGED`, and say why it is safe when you do.
   `git push origin --delete feature/<id>` may find the remote branch already gone if the
   repository auto-deletes on merge — tolerate that, it is not an error.

**Every step of phase 2 is idempotent and optional.** Between the two phases a human closes
sessions, and the harness asks whether to keep or remove the worktree on exit — so the *usual* case
by the time phase 2 runs is that the worktree is already gone, the branch may be gone too, and the
session may already be on the default branch. Each step checks, acts if there is something to do,
and reports what was already done. Phase 2 never fails because something already happened.

**When phase 2 refuses:**

| Situation | Response |
|---|---|
| PR still open | Report the state and stop — there is nothing to clean yet |
| Worktree holds unpushed work | Stop and list it |
| PR closed without merging | Not automatic cleanup: ask explicitly, because deleting that branch discards work that never landed |

## Degrading, rather than failing

| Missing | Behaviour |
|---|---|
| `gh` not installed or not authenticated | Commit and push, then print the `gh pr create` command for the user to run |
| No remote at all | Stop at the local commit and say so |
| Not a git repository | The plan is still written; nothing git-shaped is attempted |

## What still requires an explicit confirmation

Exactly one path: **`abandoned`**. The close commit and its cherry-pick rescue onto the default
branch are executed rather than printed, like everything else here — but each is confirmed first,
because it is the only flow whose expected outcome is that a branch carrying real work disappears.
Everything else in this lifecycle either lands work or removes something the merge already made
redundant.
