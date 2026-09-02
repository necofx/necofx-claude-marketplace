---
name: execute-master-plan
description: This skill should be used to start the multi-agent execution of a plan that decompose-plan has already phased out — it resolves the plan folder, commits the plan if it isn't committed yet, reads the Coordinator Prompt that `execute-plan.md` already contains, and adopts it, dispatching one teammate per phase a round at a time. It replaces extracting that prompt by hand and pasting it. Trigger when the user invokes `/execute-master-plan`, with or without a `<plan-folder>` or `<TICKET-ID>` argument, or asks to "execute the plan for GH-412", "start the rounds", "run the coordinator", "kick off the phases".
---

# Execute Master Plan

## Overview

Starts step 3 of the pack: the multi-agent execution of an already-decomposed plan. It exists to
delete one piece of ceremony — extracting the fenced Coordinator Prompt out of `execute-plan.md`
with an `awk` incantation and pasting it as the first message — and to commit the plan before the
run, which used to be a command you were told to run yourself.

**What it deliberately does not do is reimplement coordination.** `decompose-plan` already wrote a
Coordinator Prompt for this specific plan, with its real round structure, folder paths and plan slug
substituted in. This skill reads that block and adopts it as its own instructions. Everything it
owns for itself is short: resolve the folder, guard the context, commit the plan, hand over. The
less protocol it restates, the less there is to drift from the file that generates it.

**It cannot give itself a fresh context window, and does not pretend to.** Step 3 is the one
transition in the pack that requires a near-empty conversation, a skill has no way to measure or
clear its own context, and so the honest thing — and the thing Step 2 below does — is to state the
cost and let you decide, rather than quietly running a coordinator that will compact mid-plan.

## Inputs

- **`<plan-folder>` or `<TICKET-ID>`** (optional): a path to a plan folder, or a ticket id like
  `GH-412`. Step 1 resolves whichever form is given, or locates the folder itself when the argument
  is absent — the same input grammar `close-master-plan` uses, deliberately.

## Workflow

### Step 1 — Resolve and validate

1. Resolve the project root (nearest ancestor of CWD with a `.git` entry) and `<plans-root>` the way
   `create-master-plan`'s `references/plan-layout.md` describes — `docs/plans/` by default.
2. Resolve the plan folder from the argument. With no argument: if `<plans-root>/active/` holds
   exactly one folder, propose it and confirm; if several, ask via `AskUserQuestion`; if it is empty
   or absent, fall back to flat-legacy folders under `<plans-root>/` (excluding `active/`, `closed/`
   and `INDEX.md`) under the same rule.
3. **Require `execute-plan.md` and `tasks.md` to both exist.** Without them this plan was never
   decomposed: say exactly that, point at `/decompose-plan <folder>`, and stop. Do not try to
   coordinate a plan that has no phases.
4. If the folder is under `<plans-root>/closed/`, this plan is already closed — report its status
   header and stop.

**Warn, without blocking, when the session looks like the wrong checkout.** If the plan folder shows
no history on this branch and the branch name does not reference the ticket id, say so: the pack's
most common mistake is running from the main checkout while the work lives in the worktree
`create-master-plan` created. Every path below is relative to wherever this session is actually
running.

### Step 2 — The fresh-context guard

Do this **before** reading the plan files, so a wrong answer costs nothing.

Ask once, via `AskUserQuestion`, whether this conversation is fresh — and state the real cost rather
than asking an abstract question. The coordinator ends up holding the master plan, every phase file
and every teammate's report simultaneously. Started in a window that already contains a planning
discussion, it compacts mid-run, and a coordinator that has forgotten round 1's deviations
dispatches round 2 straight on top of them.

Two outcomes, and **the default is to stop**:

- *This conversation is fresh* → continue to Step 3.
- *Stop — I'll open a new window* → stop immediately, printing the one-line command to run there:
  `/execute-master-plan <folder>`. Nothing has been changed at this point, so there is nothing to
  undo.

Stopping costs one keystroke. Being wrong costs the run.

### Step 3 — Commit the plan

The plan folder must be committed before agents start writing code — otherwise the first batched
commit mixes the plan with the implementation and nobody can review either.

1. **Check the index first.** If anything is already staged *outside* the plan folder
   (`git diff --cached --name-only`), stop and list it. `git commit` includes everything staged, so
   an abandoned `git add` would otherwise land inside the plan commit without anyone asking for it.
2. `git add <plan-folder>` — the explicit path, never `-A`.
3. If that staged nothing, the plan is already committed and clean: skip in silence. This is the
   normal case when you committed after decomposing, not an error.
4. Otherwise commit as `<TICKET-ID>: plan`.

Do not push, and do not touch any branch. Pushing belongs to `close-master-plan`, at the end.

### Step 4 — Read the Coordinator Prompt out of `execute-plan.md`

Parse **structurally, never by line offsets**:

1. Find the `## Coordinator Prompt` heading. Tolerate heading-level and capitalisation variants the
   way the rest of the pack tolerates filename variants.
2. Take the **first fenced code block after that heading**. That block is the prompt.

Three ways this can go wrong, and what each one does:

- **No such heading, or no fenced block under it.** Do not guess and do not reconstruct a prompt of
  your own. Print the path to `execute-plan.md`, say which part is missing, and tell the user to
  paste the block by hand — which is exactly the flow this skill replaces, so nothing is lost but
  the convenience.
- **The block still contains unsubstituted `{PLACEHOLDER}` tokens.** That is a defect in the file
  `decompose-plan` generated, not something to patch over. Report the tokens found and stop. A
  coordinator run against a prompt that says `{FOLDER}` fails later and more confusingly.
- **The block references phase files that do not exist.** Report the mismatch and stop; the
  decomposition and the folder have diverged.

> **This heading-plus-fenced-block shape is a contract**, not an implementation detail. It is
> produced by `decompose-plan`'s `references/execute-plan-template.md` and consumed here, from a
> different plugin with its own version number. Changing that shape on either side is a breaking
> change and needs both sides updated together.

### Step 5 — Detect a resume

Read `tasks.md` before starting. If any phase is already `in_progress` or `completed`, this is a
**resume**, not a fresh start — a run that died mid-round is one of the most common things that
happens here.

No new logic is needed: the Coordinator Prompt's own loop already picks the lowest-numbered round
with `pending` phases. What is needed is **saying so**. Report which phases are done, which are in
flight, and which round the run will therefore resume at, and confirm before dispatching. A
coordinator that silently starts on a half-finished plan is indistinguishable from one that lost
its memory.

A phase stuck in `in_progress` with no live teammate is the usual wreckage of an interrupted run.
Surface it and ask whether to re-dispatch it or treat it as done, rather than assuming either.

### Step 6 — Adopt the prompt and coordinate

Adopt the extracted block as your instructions and follow it exactly — including its own setup
sequence (`superpowers:using-superpowers`, `superpowers:subagent-driven-development`, and
`superpowers:dispatching-parallel-agents` when a round has more than one phase), its reading order,
its per-round dispatch loop, its end-of-round summaries, and its blocker handling.

**Do not paraphrase, reorder, improve or abbreviate it.** It was generated for this plan with its
round structure and file-conflict verdict already resolved. Two rules from it are the ones most
often lost when someone "helps": every phase in a round is dispatched in a **single message**, one
`Agent` call each — sequential dispatch forfeits the entire point of rounds — and **phases never
commit**, because commits are batched by the coordinator at the end of a round.

From here on, this skill is finished and the prompt is in charge. The batched commits stay what
they are today: surfaced to the user at the end of each round, not run silently on their behalf.

## Principles

- **Own as little protocol as possible.** Resolve, guard, commit, hand over. Everything else lives
  in the file `decompose-plan` writes, which is the only copy that knows this plan's shape.
- **The fresh-context requirement is real and cannot be enforced from inside.** Refusing to ignore
  it is the whole of what this skill can honestly do about it.
- **Degrade to the manual path, never to a guess.** Every failure mode ends either in "paste it
  yourself, here is the path" or in a named defect — never in a coordinator improvising a prompt.
