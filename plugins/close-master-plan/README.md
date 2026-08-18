# close-master-plan

Step 6 of a plan-first multi-agent workflow: closes out a master implementation plan once its work has landed on a branch. It reconciles `tasks.md` against the real commits, verifies `handoff.md` is complete, distils the run's durable lessons into `.claude/rules/`, stamps the plan's outcome, and archives the folder under `docs/plans/closed/`.

**It never changes git state itself.** Every step that touches files stages them; the skill stops short of committing, pushing, switching branches, or touching the worktree it is running inside. It prints the exact commands, and you run them.

## When to run it

After `/code-review` inside Claude Code and the optional `/plan-implementation-review` from [`plan-review`](../plan-review/) — this is the last step in the pack, not a substitute for either review. Run it once the branch's work is actually done (or actually abandoned), before merging or finishing the branch. `superpowers:finishing-a-development-branch` picks up from there: merging, pushing, and worktree removal are not this skill's job.

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

What it does, in order:

1. **Locate and validate.** Checks `closed/<ID>/` first — if the plan is already there, it reports the existing status and stops rather than re-closing it, which is what keeps a second run idempotent. Otherwise finds the plan under `active/<ID>/` or the flat legacy layout, and requires `master-plan.md` to exist.
2. **Git preflight.** The working tree must be clean — a close commit that also contains unfinished work defeats the point of the archive. Resolves the base branch and the branch's commit list, and warns (without blocking) if the run looks like it's happening from the wrong checkout, or if no implementation-review artifact is present in the folder.
3. **Establish the status**, via `AskUserQuestion`: `completed`, `abandoned`, or `superseded by <TICKET-ID>`. There is no `merged` status — at the point this runs, the PR hasn't merged yet, and the PR number is the durable pointer to that outcome.
4. **Reconcile `tasks.md`.** Proposes a phase-to-commit mapping built from `tasks.md`'s Detailed Progress entries and the branch's commit subjects, and has you correct it before applying it — commit messages carry no contracted format, so the mapping is never derived silently. Fills in `Finished` timestamps and a `Final Summary`. A plan can't be stamped `completed` while any phase is still `pending`, `in_progress`, or `blocked`; mark it `dropped` with a justification, or finish the phase first. (Skipped entirely if the folder has no `tasks.md` — a plan that was never decomposed can only close as `abandoned` or `superseded`.)
5. **Verify `handoff.md`.** Scans it for template placeholders left behind, and asks you to confirm out loud that an empty "Key deviations from the original plan" section really is empty — that's the section reviewers read most closely.
6. **Distil durable lessons.** Reads `handoff.md`'s deviations and `tasks.md`'s `Decisions` and `Coordination Notes`, and presents anything that would still be true on the next ticket, in a different part of the codebase, as a concrete diff against the matching `.claude/rules/*.md` file. Every candidate needs an explicit yes, one at a time — nothing is bundled, and zero rules distilled is a common, valid outcome.
7. **Stamp.** Writes a one-line status header into `master-plan.md` (and into `tasks.md` / `handoff.md`, if they exist) recording the status, the close date, the PR number, and which rules files got written.
8. **Move and index.** `git mv`s the folder to `<plans-root>/closed/<ID>/` and updates `<plans-root>/INDEX.md` with a newest-first row. Stops and lists what's there rather than overwriting if the destination already exists.
9. **Report and stop.** Prints the `git add` + commit command for you to run. For `abandoned`, it prints a second command too — see below.

---

## Two things to understand before you run it

### `abandoned` has a two-part ending

If you close a plan as `abandoned`, the skill still performs the close **on the current branch**, because the plan folder's content only exists there right now. But a commit on a branch that's about to be deleted is not durable — deleting the branch takes the archive with it, which is exactly the record this skill exists to preserve.

So for `abandoned`, Step 9 prints two commands, both required, not one optional follow-up:

```sh
git add docs/plans .claude/rules && git commit -m "GH-412: close plan (abandoned)"
# the branch is being deleted — the archive must also land on the default branch:
git switch main && git cherry-pick <that-commit>
```

Expect the cherry-pick to hit rename/delete conflicts — the plan folder usually never existed on the default branch, only on the branch being abandoned, so there's no pre-image for Git to rename from. Resolve by `git add`-ing the reported paths and running `git cherry-pick --continue`; that's the normal path, not a sign something went wrong.

### There is no reopen in v0.1.0

Closing the wrong plan, or closing with the wrong status, has no dedicated undo command. Use `git revert` on the close commit, or reverse it by hand: `git mv` the folder back to `active/`, strip the header from each file it was stamped into, and delete the row `INDEX.md` gained.

There's no automated reopen because it would have to honestly reverse three different kinds of change at once: a `.claude/rules/` edit you already approved as a standalone improvement (not scoped to this plan), a header that replaced whatever was there before (only recoverable from git history), and a `tasks.md` reconciliation you built by hand. Automating that would claim a reversibility the skill can't actually guarantee — `git revert` or a manual fix says exactly what happened, instead of pretending the close never occurred.

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
| "Already closed" and it stops | `<plans-root>/closed/<ID>/` exists. The skill reports the existing status rather than re-closing — this is what makes closing idempotent, not a bug. |
| Destination already exists at Step 8 | Something is already at `closed/<ID>/`. The skill lists what's there and stops; there's no merge option for an already-closed folder, only your judgment call. |
| Base branch is ambiguous | Asked via `AskUserQuestion` rather than guessed — diffing a committed branch against `HEAD` would vacuously find nothing to close. |
| Warned about running from the wrong checkout | The branch name doesn't reference the ticket id and the plan folder has no history on this branch — the pack's most common mistake is running from the main checkout while the real work is in a worktree. Informational, not a gate. |
| Plan folder was never committed | `git mv` has nothing to rename from. The skill falls back to a plain `mv` and says so — the move still happens, it just isn't a tracked rename. |

## Limits

- **It never touches git state beyond staging.** No commit, no push, no branch switch, no worktree removal — every command is printed for you to run yourself.
- **No reopen command.** See [above](#there-is-no-reopen-in-v010).
- **A plan with no `tasks.md` can only close as `abandoned` or `superseded`.** It was never decomposed into phases, so there's nothing to reconcile against commits or verify in `handoff.md` — rule distillation still runs off `handoff.md` alone, when present.
- **Migrating an existing flat-layout project onto `active/`/`closed/` is manual.** Both this skill and `create-master-plan` keep working with the flat layout indefinitely; nothing forces the move.
