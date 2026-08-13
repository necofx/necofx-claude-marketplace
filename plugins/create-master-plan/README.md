# create-master-plan

Turns a ticket into a researched plan. It fetches the issue with every field, comment and attachment, follows up to 5 linked issues and 3 Confluence pages, scans the repository's `docs/` exhaustively for anything that mentions the ticket, detects the tech stack, then interviews you over a coverage matrix for what none of that could answer — and writes `issue.specs` and `master-plan.md`.

It stops there. [`decompose-plan`](../decompose-plan/) phases the plan out; this skill never decomposes and never writes back to the tracker.

## Credits

**This is not our work.** The workflow was designed by someone else, who uses it daily and shared their files directly; every file under `skills/` is a byte-for-byte copy of theirs, verified with `diff -r` against the original at packaging time. Nothing here is a reimplementation or an improvement, and it is published with their permission.

What this repository adds is the packaging: a plugin manifest, this README, and `config.md`. Those are the only files here that are not theirs.

`MANUAL.html` in this folder is their manual for the complete five-step workflow. It is the best explanation of why the workflow is shaped this way, and it is worth reading before the first run.

---

## Setup

Four steps. The third is the one people skip and then wonder why the run fails.

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

### 3. Configure the placeholders — required

The skill ships with three unresolved `{{TOKEN}}` placeholders that a plugin install cannot substitute, because an installed plugin lives in a read-only cache under `~/.claude/plugins/`. You supply the values from the repository you are planning in instead.

**Paste this into that repository's `CLAUDE.md`**, which Claude Code loads at the start of every session:

```markdown
## Planning workflow configuration

The `create-master-plan` and `decompose-plan` skills carry unresolved `{{TOKEN}}`
placeholders. Resolve them with these values wherever they appear, and never pass a
literal `{{...}}` to a tool:

- `{{PLANS_DIR}}` → `docs/plans`
- `{{TICKET_PREFIX}}` → `GH`
- `{{JIRA_CLOUD_ID}}` → not applicable; this project tracks work in GitHub Issues.

### Ticket source: GitHub, not Jira

Ignore the skill's Atlassian MCP steps (Step 2 fetch, Step 4 linked issues and
Confluence pages) and substitute the `gh` CLI, keeping every downstream step and the
`issue.specs` structure exactly as written:

- Base issue: `gh issue view <N> --json number,title,body,state,labels,assignees,author,comments,milestone`
- Linked work: follow the `#<n>` and PR references in the body and comments, honouring
  the skill's cap of 5 linked issues.
- Referenced PRs: `gh pr view <N> --json title,body,files,state,mergedAt` — the `files`
  list is the highest-value field for grounding the plan in what already landed.
- Attachments: download the `github.com/user-attachments` URLs found in the body and
  comments into `<plan-folder>/attachments/`.

The ticket id is `GH-<issue number>` (e.g. `GH-412`), which satisfies the skill's
`^[A-Z]+-[0-9]+$` validation, and the plan folder is `docs/plans/GH-<number>/`.
```

Using Jira instead? [`config.md`](config.md) has that variant, plus how to change any of the three defaults. No tracker at all? Also `config.md` — you hand-write `issue.specs` and start at `/decompose-plan`.

### 4. Optional, and both are genuinely optional

**Atlassian MCP**, if you use Jira and want the original fetch path rather than the GitHub adaptation:

```sh
claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp
claude mcp list          # expect: atlassian - Connected
```

First use opens a browser for OAuth. Set `{{JIRA_CLOUD_ID}}` to the bare site host — `your-org.atlassian.net`, no `https://`, no path.

**CodeGraph**, which the interview uses to ground its questions in the code before asking:

```sh
npm install -g @colbymchenry/codegraph    # a real global install; npx-only silently fails
codegraph install -y -t claude -l global  # the agent id is `claude`, not `claude-code`
codegraph telemetry off                   # telemetry ships ENABLED
cd /path/to/your/repo && codegraph init
```

Every CodeGraph instruction in the skill is conditional on a `.codegraph/` directory existing, so an unindexed repo simply costs more tokens. **Indexing is your decision and the skill never does it for you** — it writes hundreds of megabytes.

### 5. Restart

Skills load at session start. Open a new conversation before your first `/create-master-plan`.

---

## Tutorial: your first master plan

Pick a small, well-understood ticket. You are testing plumbing, not ambition — that the fetch connects, that the interview asks sensible questions, that the plan is one you would have written yourself.

### Run it

```
/create-master-plan GH-412
```

Takes a bare ticket key, or a full Atlassian URL it will parse the key out of. Omit the argument and it asks.

### What happens, and where you are involved

**It creates the plan folder** at `docs/plans/GH-412/`. If that folder already exists with files in it, the skill stops and asks: overwrite the specific files (typically `issue.specs` and `master-plan.md`, preserving `attachments/` and any `phases/`), merge (renaming the old ones to `.bak.<timestamp>`), or abort. It will not silently clobber anything.

**It fetches and reads, and you wait.** The issue with all fields and comments, attachments downloaded into `attachments/`, up to 5 linked issues in priority order (parent epic → blocks → relates-to → sub-tasks), up to 3 Confluence pages. Then it globs `docs/**/*.md` with **no exclusions and no cap** and greps every file for the ticket key, the component names and the distinctive nouns from the summary. Long output here is expected and correct — this is the step that surfaces the internal document nobody remembered.

A failed attachment download or an unreachable Confluence page is recorded in `issue.specs § Context Gaps` and never aborts the run.

**It writes `issue.specs` before asking you anything.** Nine sections: description, acceptance criteria, comments, linked issues, Confluence, attachments, related local docs, context gaps. Read it if you want, but the ordering is the point — everything in that file now counts as known, so a question the interview asks that the file already answers is a defect you can point at.

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
/decompose-plan docs/plans/GH-412
```

**In a new conversation.** This one ends with the full ticket thread, every doc excerpt and the whole interview in context, and the next step needs the room. The plan is on disk; this conversation has no further value.

---

## Limits

- **The placeholders are resolved by context, not by substitution.** If a run ever passes a literal `{{...}}` to a tool, your `CLAUDE.md` block is missing or was not loaded. That is the first thing to check.
- **It is Jira-shaped.** `MANUAL.html` says step 1 is the only step with a tracker dependency. The GitHub adaptation redirects the fetch; everything downstream reads `issue.specs` and does not care where it came from.
- **It never writes to the tracker.** Read-only fetches only: no comments, no transitions, no label edits.
- **It never commits.** The files are yours to review and commit.
