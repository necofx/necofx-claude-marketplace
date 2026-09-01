# create-master-plan

Forks a worktree for the ticket, then turns that ticket into a researched plan. It fetches the issue with every field, comment and attachment, follows up to 5 linked items and 3 cited documents, scans the repository's `docs/` exhaustively for anything that mentions the ticket, detects the tech stack, then interviews you over a coverage matrix for what none of that could answer — and writes `issue.specs` and `master-plan.md`.

It stops there. [`decompose-plan`](../decompose-plan/) phases the plan out; this skill never decomposes and never writes back to the tracker.

## Credits

**The workflow is not ours.** It was designed by someone else, who uses it daily and shared their files directly, and it is published here with their permission. The structure, the reasoning and nearly all of the prose are theirs.

**What we changed, and it is one thing:** the original fetches from Jira and carries template placeholders you substitute at install time. This package makes **GitHub the default** and moves the tracker behind an adapter layer (`references/source-adapters.md`), so Jira and Linear are profiles rather than the hard-wired path, and the placeholders are gone. Steps 3-9 were left alone — they read a normalised record and never knew which tracker ran.

`MANUAL.html` in this folder is their manual for the complete six-step workflow, updated to match. It is the best explanation of why the workflow is shaped this way, and it is worth reading before the first run.

---

## Setup

Five steps, and on GitHub only two of them are work.

### 1. Install the plugin

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install create-master-plan@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH — pass `https://github.com/necofx/necofx-claude-marketplace.git` instead.

### 2. Install `superpowers` — required

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install superpowers@claude-plugins-official
```

The skill invokes `superpowers:using-superpowers` and `superpowers:writing-plans` before writing the master plan. Without them it falls back to its own `references/master-plan-template.md` and records the fallback in the generated file — the run survives, the plan-shaping discipline does not.

### 3. Nothing to configure — unless you are not on GitHub

**GitHub is the default and needs no setup.** The skill uses the `gh` CLI (make sure `gh auth status` is happy), derives the plan id as `GH-<issue number>`, and writes to `docs/plans/active/GH-<number>/`.

Using **Jira** or **Linear**, or want a different plans directory? That is what [`config.md`](config.md) is for — one MCP server and three lines in your repository's `CLAUDE.md`. No tracker at all is also fine: paste the requirement and the skill takes its free-form adapter.

### 4. Optional, and genuinely optional

**CodeGraph**, which the interview uses to ground its questions in the code before asking:

```sh
npm install -g @colbymchenry/codegraph    # a real global install; npx-only silently fails
codegraph install -y -t claude -l global  # the agent id is `claude`, not `claude-code`
codegraph telemetry off                   # telemetry ships ENABLED
cd /path/to/your/repo && codegraph init
```

Every CodeGraph instruction in the skill is conditional on a `.codegraph/` directory existing, so an unindexed repo simply costs more tokens. **Indexing your repository is your decision and the skill never does it for you** — it writes hundreds of megabytes. The one thing it does build unasked is an index *inside the ticket's worktree*, and only when the parent repository already has one: that copy is disposable, and it disappears when `close-master-plan` removes the worktree.

### 5. Restart

Skills load at session start. Open a new conversation before your first `/create-master-plan`.

---

## Tutorial: your first master plan

Pick a small, well-understood ticket. You are testing plumbing, not ambition — that the fetch connects, that the interview asks sensible questions, that the plan is one you would have written yourself.

### Run it

```
/create-master-plan GH-412
```

Takes a GitHub issue number (`412`, `#412`, or an issue URL), or a tracker key like `ACME-1234` if you configured Jira or Linear. Omit the argument and it asks.

### What happens, and where you are involved

**First it forks into a worktree for the ticket** — `EnterWorktree` on the ticket id, then a rename of the harness's `worktree-gh-412` branch to `feature/gh-412` before anything is pushed. Your session's working directory moves there, and everything from here on is written inside that worktree, on that branch. The report at the end tells you where you ended up; it is the one thing you should never have to discover for yourself.

This happens **before** the plan folder is created, and the ordering is load-bearing rather than stylistic: the folder is written into whichever tree the session is standing in, so forking afterwards would leave it in your main checkout while the worktree — a clean checkout of the new branch — doesn't contain it.

Four situations skip the fork silently and continue: you're already inside a worktree (it stays there), the directory isn't a git repository, there's no `origin` to branch from, or the harness can't move the session. None of them stop the run, and none of them ask.

**If the repository already has a `.codegraph/` index, the worktree gets its own** — `codegraph init` starts in the background while the ticket fetch and the `docs/` scan proceed, so it's usually ready by the time the interview needs it. This is the single exception to the pack's "never index a repository yourself" rule, and it's narrow on purpose: a parent index means you already opted into CodeGraph here, and the worktree's copy lives inside the worktree, self-gitignores, and disappears when [`close-master-plan`](../close-master-plan/) cleans up. **A repository with no index gets none** — nothing changes for you.

Without a local index CodeGraph still answers from the parent's, warning that results reflect a different working tree. At plan time that's harmless, because the worktree is still identical to its base; the drift only appears once agents start writing code.

**Then it creates the plan folder** at `docs/plans/active/GH-412/`. If that folder already exists with files in it, the skill stops and asks: overwrite the specific files (typically `issue.specs` and `master-plan.md`, preserving `attachments/` and any `phases/`), merge (renaming the old ones to `.bak.<timestamp>`), or abort. It will not silently clobber anything. (If your project is still on the flat legacy layout — a plan folder directly under `docs/plans/<TICKET-ID>/`, no `active/`/`closed/` split — the skill uses that folder unchanged and says so; it never migrates a project onto the new layout on its own.)

**It picks the source adapter and fetches.** GitHub unless the detection ladder says otherwise; where two adapters match, it asks rather than guessing. Then: the issue with all fields and comments, attachments downloaded into `attachments/` (and images actually read), up to 5 linked items in priority order (parent epic → blocks → relates-to → sub-tasks), up to 3 cited documents quoted in full rather than linked. **A referenced merged PR is the most valuable thing it finds** — its file list is how "this ticket is gap-closure, not a build" gets discovered without grepping blind. Then it globs `docs/**/*.md` with **no exclusions and no cap** and greps every file for the ticket key, the component names and the distinctive nouns from the summary. Long output here is expected and correct — this is the step that surfaces the internal document nobody remembered.

A failed attachment download or an unreachable document is recorded in `issue.specs § Context Gaps` and never aborts the run. So is a field your source has no equivalent for — GitHub has no components or priority, and saying so beats inventing them out of labels.

**It writes `issue.specs` before asking you anything.** Nine sections: description, acceptance criteria, comments, linked issues, referenced documents, attachments, related local docs, context gaps. Read it if you want, but the ordering is the point — everything in that file now counts as known, so a question the interview asks that the file already answers is a defect you can point at.

**Then the interview starts, and this is the part that decides quality.** Every question arrives through `AskUserQuestion` with real options, batched up to four at a time. It works through a coverage matrix: goal and value, constraints, scope boundary, technical implementation, UI/UX where a user-facing surface exists, edge cases, tradeoffs, acceptance criteria, testing strategy, validation gates. It skips anything already answered, and it grounds each question in the code first.

Answer properly. Everything downstream derives from this — a vague requirement here becomes several agents confidently building the wrong thing in parallel two steps later. Expect several rounds; two rounds only covers the obvious.

**Before writing, it shows you the outline** and asks to confirm: *looks good — write it* / *edit the outline first* / *ask more questions*. Use the middle option freely, it is cheaper than fixing the plan afterwards.

**It writes `master-plan.md`** and reports back: the folder path, files written with sizes, how much context it fetched, the top related docs, how many interview rounds, and the next command.

### What you get

| File | Contains |
|---|---|
| `issue.specs` | Everything fetched, in a stable 9-section structure, plus `## Interview Notes` appended with each Q&A — so a later run sees what was already settled. |
| `master-plan.md` | Context, tech stack, pre-flight checklist, why, out of scope, technical requirements by layer, implementation outline, test configuration, validation gates, acceptance criteria. |
| `attachments/` | Downloaded ticket attachments, one per file, with failures logged in Context Gaps. |

### Check these two things before moving on

**Is the implementation outline at chunk level?** It should stop at "add the X endpoint", not name functions. Pre-decomposing here produces phases that collide later.

**Do the validation gates name your real commands?** If the plan says `npm test` and your project actually needs `npm test -- --run`, fix it now. An assumed verification command does not fail at planning time — it fails in round 2 of the multi-agent execution, when four phases already rest on it and correcting it means redoing the decomposition.

### Next

```
/decompose-plan docs/plans/active/GH-412
```

**In a new conversation.** This one ends with the full ticket thread, every doc excerpt and the whole interview in context, and the next step needs the room. The plan is on disk; this conversation has no further value.

---

## Limits

- **Step 1 is the only step with a tracker dependency at all.** Everything downstream reads `issue.specs` and does not care where it came from, which is why swapping the source is a one-file change and why you can skip this step entirely by hand-writing the dossier.
- **It never writes to the tracker.** Read-only fetches only: no comments, no transitions, no label edits.
- **It never commits.** It creates a worktree and a branch — which write no history and touch nothing that already existed — and stops there. The files are yours to review and commit. Committing, pushing and opening the PR happen at the other end of the lifecycle, in [`close-master-plan`](../close-master-plan/); the contract both ends share is [`git-lifecycle.md`](skills/create-master-plan/references/git-lifecycle.md), carried byte-identically by both plugins.
- **It indexes nothing you hadn't already chosen to index.** The worktree's CodeGraph index is built only when the parent repository already has one, and it dies with the worktree.
