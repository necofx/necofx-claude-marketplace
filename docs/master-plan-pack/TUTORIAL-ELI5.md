# The master-plan pack, explained like you're five

Build code the way you build a house. First you draw the plan. Then you split the work. The crews build. An inspector checks. You hand over the keys.

For the full worked example, with every command and artifact, read [`TUTORIAL.md`](TUTORIAL.md). This page is the picture version.

```
           ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
 ticket ─▶ │  1 CREATE   │─▶ │ 2 DECOMPOSE │─▶ │  3 EXECUTE  │─▶ │  4 REVIEW   │─▶ │   5 CLOSE   │ ─▶ merged
           │  the plan   │   │  the tasks  │   │  the work   │   │  the work   │   │   the PR    │
           └─────────────┘   └──────┬──────┘   └─────────────┘   └─────────────┘   └─────────────┘
                                    │
                                    └─▶ 2½ REVIEW the plan (optional)
```

---

## 1 · Draw the plan

```
      ┌──────────────────┐        (o)   "what if…?"
      │  ┌──────┬──────┐ │        /|\
      │  │      │      │ │        / \
      │  ├──────┤      │ │
      │  └──────┴──────┘ │      the architect asks
      └──────────────────┘      until it understands
          master-plan.md
```

Give it a ticket number. It reads the ticket, its links and your docs. It asks you questions until it understands. Then it writes the plan: what gets built, what was decided, and what must be tested no matter what.

```
/create-master-plan 412
```

Creates the plan folder (and a worktree): `issue.specs` · `master-plan.md`

---

## 2 · Cut the plan into tasks

```
                         round 1:  [P1]  [P2]  [P3]   ← all at once
   ┌──────────┐
   │   plan   │  ──✂──▶
   └──────────┘
                         round 2:  [P4]  [P5]         ← after round 1
```

It splits the plan into small phases, one per worker. Phases that don't touch the same files go together in the same **round**. Each phase says what it delivers, what it needs from other phases, and which test proves it works.

```
/decompose-plan docs/plans/active/GH-412-…
```

Adds `PHASE-01…md` · `tasks.md` · `execute-plan.md` · `handoff.md`

---

## 2½ · Get a second opinion on the plan (optional)

```
      ┌──────────────────┐
      │  ┌──────┬──────┐ │      ╭───╮
      │  │      │    ( ? )──────┤   │  a second architect
      │  └──────┴──────┘ │      ╰───╯  looks for holes
      └──────────────────┘
```

Optional, but cheap. It writes a message for Codex (another model) to review the plan **before** anything is built. You paste that message into Codex and fix whatever it finds.

```
/plan-review-prompt <plan-folder>
```

---

## 3 · The foreman starts the job

```
                       ┌─▶  (o) worker  ── P1
        (o)  foreman   │
        /|\ [plan] ────┼─▶  (o) worker  ── P2      one round,
        / \            │                           sent together
                       └─▶  (o) worker  ── P3
```

Open a **new window** and call it. It commits the plan to git and sends one worker per phase. All the workers in a round start at the same time. When a round ends, it tells you what happened and proposes a commit.

```
/execute-master-plan GH-412
```

If the run stops halfway, call it again. It picks up at the round that is still pending.

---

## 4 · The inspector checks the work

```
      ┌──────────┐            /\
      │ ┌──────┐ │    ==     /  \        ✔
      │ └──────┘ │          │ [] │
      └──────────┘          └────┘
         the plan        what got built
```

It writes another message for Codex. This one compares the real code against the plan. Is anything missing? Is anything extra? Are the tests the plan asked for actually there? And on top of that: is the code well made?

```
/plan-implementation-review <plan-folder>
```

---

## 5 · Hand over the keys

```
      ┌─────────────┐
      │▓▓▓▓▓▓▓▓▓▓▓▓▓│          ○━━━━┳┳
      │   closed/   │        the keys
      └─────────────┘
```

**First run:** it checks that everything is done, archives the plan, pushes the branch and opens the PR. **You** decide the merge. **Second run**, once the PR is merged: it deletes the branch and the worktree.

```
/close-master-plan GH-412
```

The plan folder moves from `docs/plans/active/` to `docs/plans/closed/`.

---

## Three rules of the job site

1. **The foreman needs a new window.** It holds the plan, every phase and every worker's report. In a conversation that is already full, it forgets things halfway through.
2. **Workers never commit.** Each one does its phase and hands in a report. The foreman batches the commits at the end of each round.
3. **The merge is yours.** No plugin merges. Close opens the PR and waits for you to approve it.

---

## Cheat sheet

| When | Command | You get |
|---|---|---|
| You have a ticket | `/create-master-plan 412` | the plan |
| The plan looks right | `/decompose-plan <plan-folder>` | phases and rounds |
| You want a second opinion on the plan | `/plan-review-prompt <plan-folder>` | a message for Codex |
| New window, time to build | `/execute-master-plan GH-412` | code, round by round |
| The code is done | `/plan-implementation-review <plan-folder>` | a message for Codex |
| Everything is reviewed | `/close-master-plan GH-412` | an open PR |
| The PR is merged | `/close-master-plan GH-412` | branch and worktree deleted |
