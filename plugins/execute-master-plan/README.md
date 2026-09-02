# execute-master-plan

Step 3 of a plan-first multi-agent workflow: starts the execution of a plan [`decompose-plan`](../decompose-plan/) has already phased out. It resolves the plan folder, commits the plan if it isn't committed yet, reads the Coordinator Prompt that `execute-plan.md` already contains, and adopts it — then dispatches one teammate per phase, a round at a time.

**It replaces one thing: extracting that prompt by hand.** Until now step 3 meant opening `execute-plan.md`, selecting everything between the two fences under `## Coordinator Prompt`, and pasting it as your first message — or running an `awk` incantation to lift it out. That is the ceremony this deletes.

`MANUAL.html` in this folder documents the complete six-step workflow this plugin sits in the middle of.

## What it does not do

**It does not reimplement coordination.** The Coordinator Prompt is generated per plan by `decompose-plan`, with that plan's real round structure, folder paths and slug already substituted in. This skill reads that block and adopts it. Everything it owns for itself is short — resolve, guard, commit, hand over — because every line of protocol it restated would be a line that can drift from the file that generates it.

**It cannot give itself a fresh context window.** Step 3 is the one transition in this pack that genuinely requires a near-empty conversation: the coordinator ends up holding the master plan, every phase file and every teammate's report at once, and one started in a window that already contains your planning discussion will compact mid-run. A skill has no way to measure or clear its own context, so this one asks you once, up front, and **defaults to stopping** — see below.

**It does not push, branch, or merge.** The only git it runs is the plan commit. Pushing and the PR belong to [`close-master-plan`](../close-master-plan/), at the end.

---

## Setup

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install execute-master-plan@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH — pass `https://github.com/necofx/necofx-claude-marketplace.git` instead.

Install [`superpowers`](https://github.com/anthropics/claude-plugins-official) too — the Coordinator Prompt invokes `superpowers:using-superpowers`, `superpowers:subagent-driven-development` and `superpowers:dispatching-parallel-agents` as its first act:

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install superpowers@claude-plugins-official
```

Then start a new conversation — skills load at session start.

---

## Usage

**Open a fresh conversation first.** Then:

```
/execute-master-plan [<plan-folder> | <TICKET-ID>]
```

Give it a path, a ticket id like `GH-412`, or nothing — with no argument it looks in `<plans-root>/active/`, proposes the one folder it finds there (or asks which, if there are several), and falls back to a flat legacy layout under the same rule. Same input grammar as `/close-master-plan`, deliberately.

What happens, in order:

1. **Resolve and validate.** Requires `execute-plan.md` and `tasks.md` to exist — without them the plan was never decomposed, and it points you at `/decompose-plan` instead of trying to coordinate a plan with no phases. Warns (without blocking) if the session looks like the wrong checkout.
2. **The fresh-context question.** One `AskUserQuestion`, before anything is read or changed, stating the actual cost rather than asking abstractly. **The default is to stop.** Stopping costs one keystroke; being wrong costs the whole run.
3. **Commit the plan.** `git add <plan-folder>` — the explicit path, never `-A` — then commit as `<TICKET-ID>: plan`. If something is already staged *outside* the plan folder it stops and lists it, because `git commit` would otherwise sweep it in. Nothing staged means the plan is already committed: it skips in silence.
4. **Read the Coordinator Prompt** out of `execute-plan.md`, structurally: the `## Coordinator Prompt` heading, then the first fenced block after it. Never line offsets.
5. **Detect a resume.** If `tasks.md` shows phases already `in_progress` or `completed`, it says so and confirms before dispatching, rather than silently starting on a half-finished plan.
6. **Adopt the prompt and coordinate** — its setup sequence, its per-round dispatch loop, its end-of-round summaries, its blocker handling, unmodified.

---

## Two things worth knowing

### Why it asks whether your conversation is fresh

Because it cannot find out. There is no API for a skill to measure its own context window and none to clear it, so the requirement that made step 3 the pack's one load-bearing transition can't be enforced from inside a skill — only respected. The question is asked first, before any file is read, so answering "stop, I'll open a new window" costs nothing and leaves nothing to undo.

If you want the guard without the question, the answer is the same as it always was: open the new conversation and make `/execute-master-plan` its first message.

### The prompt block is a contract between two plugins

`decompose-plan` writes the `## Coordinator Prompt` heading and its fenced block; this plugin reads them, from a separate plugin with its own version number. That gap is where drift happens, so the parsing is defensive and every failure is loud:

| What's wrong | What it does |
|---|---|
| No heading, or no fenced block under it | Prints the path and tells you to paste it by hand — the pre-existing flow, so you lose the convenience and nothing else |
| The block still has `{PLACEHOLDER}` tokens in it | Stops and names them: that's a defect in what `decompose-plan` generated, not something to patch over |
| The block references phase files that don't exist | Stops — the decomposition and the folder have diverged |

None of them end in a coordinator improvising a prompt of its own.

---

## Limits

- **It never merges, pushes, or creates branches.** The plan commit is the only git it runs.
- **It cannot guarantee the fresh context it depends on** — it can only refuse to ignore the requirement.
- **It coordinates the plan as written.** If the decomposition is wrong, this runs the wrong decomposition faithfully; fix the plan, not the run. `/plan-review-prompt` from [`plan-review`](../plan-review/) exists for exactly that check, before you get here.
