# close-master-plan — design

**Date:** 2026-08-18
**Status:** approved design, revised after external review, not yet implemented
**Repo:** necofx-claude-marketplace

## 1. The problem

The master-plan pack has five steps and no sixth. `TUTORIAL.md:959-965` ("Part 5 · Close out")
describes three things a human is supposed to do by hand once the last round lands:

1. Put the real commit SHAs and a final summary into `tasks.md`, replacing every `(pending batch)`.
2. Fill every section of `handoff.md`, especially *deviations from the plan*.
3. Fold anything durable into `.claude/rules/` rather than leave it "in a plan folder nobody reopens".

Nothing automates them, so in practice they are skipped. Two consequences compound:

- **Plan folders accumulate undifferentiated.** `create-master-plan` Step 5 (`SKILL.md:91-105`)
  globs `docs/**/*.md` with no exclusion, no depth cap, one hit is enough, and the *Related local
  docs* section of the new `issue.specs` is explicitly forbidden from truncating. A plan that was
  abandoned two quarters ago therefore enters the next plan's dossier with the same textual weight
  as live architecture docs.
- **The durable lesson never leaves the folder.** The one artifact worth keeping — why the code
  diverged from the plan — stays where only an accidental grep will find it.

This design adds the missing step and makes the accumulated history cheap to carry.

## 2. Scope

### In scope

- A new plugin, `close-master-plan`, that performs the close-out and archives the plan folder.
- A new plan-folder layout: `<plans-root>/active/<TICKET-ID>/` and `<plans-root>/closed/<TICKET-ID>/`,
  where `<plans-root>` defaults to `docs/plans/`.
- The changes to `create-master-plan` that the layout requires (write path, ranking closed plans
  separately in the `docs/` scan, and the section-numbering fix §5.4 uncovered along the way).
- The documentation surface across the repo that the layout invalidates.

### Explicitly out of scope

- **Migrating anybody's existing plans.** `create-master-plan` keeps working with the flat
  `docs/plans/<TICKET-ID>/` layout when it finds it, and `close-master-plan` can close a flat plan
  (§4.2 Step 8). Moving the rest is the user's call.
- **Changing `decompose-plan` or `plan-review` behaviour.** They take the plan folder as an
  argument and do not care what it is called. Only their prose examples change.
- **Any network call.** No `gh`, no PR comment, no issue close. The step is local.
- **Touching git state.** The skill does not commit, does not switch branches, does not remove a
  worktree. It writes files, stages nothing, and prints the commands.
- **Reopening a closed plan.** Cut from v0.1.0 — see §9.

## 3. Layout and contracts

### 3.1 Directory layout

```
docs/plans/                 # <plans-root>
  INDEX.md                  # closed plans only, one line each
  active/
    GH-500/                 # a plan being worked
  closed/
    GH-412/                 # completed
    GH-388/                 # superseded by GH-412
```

**`<plans-root>` is a root, not a per-ticket pattern.** Today `config.md:52-62` documents the knob
as a complete path — *"Plan folders live under `docs/prps/<TICKET-ID>/`"* — which under the new
layout is ambiguous: is `docs/prps` the root, or the parent of `active`? The rule is:

- A declaration naming a directory (`docs/prps/`) is the root. Plans go in `docs/prps/active/<ID>/`.
- A declaration that still ends in `<TICKET-ID>` (the pre-0.4.0 wording) has that final segment
  stripped, and the remainder is the root. This keeps existing `CLAUDE.md` declarations working.

`config.md` is rewritten in 0.4.0 to declare a root, and both plugins read the same value — the
constraint `config.md:62` already states, now spanning four plugins.

### 3.2 The status header — the contract between the two plugins

Archived plans carry a status comment in the **first five lines** of `master-plan.md`, and of
`tasks.md` and `handoff.md` when those exist:

```markdown
<!-- STATUS: completed · closed 2026-08-18 · PR #77 · rules: .claude/rules/java.md -->
# GH-412 — Partial refunds
```

Fields, in this order, separated by ` · `:

| Field | Values | Notes |
|---|---|---|
| status | `completed` \| `abandoned` \| `superseded by <TICKET-ID>` | Describes the **plan**, not the PR |
| closed | `closed YYYY-MM-DD` | Absolute date, never relative |
| PR | `PR #<n>` or `PR —` | Whatever the user supplies or `tasks.md` already holds |
| rules | `rules: <paths>` or `rules: —` | Comma-separated, the files Step 6 wrote to |

**There is no `merged` status.** When the skill runs, the PR has not merged — that is the whole
point of running it inside the branch so the archive travels in the PR. The status describes the
plan's own outcome; whether the PR merged is GitHub's business, and the PR number is the pointer.

The header is an HTML comment so it renders as nothing on GitHub while staying greppable, and it
sits above the H1 so `head -5` finds it without parsing.

**`master-plan.md` is the authoritative carrier.** `issue.specs`, `phases/*.md` and every other
file in the folder are *not* stamped. A consumer that matches any file under `closed/<ID>/` reads
the status from `closed/<ID>/master-plan.md`. One lookup per plan folder, regardless of how many
files matched — which is also what collapses N matches into one reported line (§5.2). If that file
is missing or carries no header (a folder moved by hand), the plan is reported with status
`unknown` rather than skipped.

### 3.3 INDEX.md

Closed plans only. Active plans are visible with `ls active/`. This keeps `close-master-plan` the
single writer of the index and spares `create-master-plan` a third change.

```markdown
# Closed plans

| Plan | Title | Status | Closed | PR | Rules distilled |
|---|---|---|---|---|---|
| GH-412 | Partial refunds | completed | 2026-08-18 | #77 | `.claude/rules/java.md` |
| GH-388 | Idempotency filter | superseded by GH-412 | 2026-06-02 | #61 | — |
```

Newest first. The file is created on first close if absent. It is a convenience for humans; nothing
in either plugin's behaviour depends on it being accurate.

## 4. Deliverable 1 — the `close-master-plan` plugin

```
plugins/close-master-plan/
  .claude-plugin/plugin.json          # v0.1.0
  README.md
  MANUAL.html                         # the shared manual, updated for six steps and the new layout
  skills/close-master-plan/
    SKILL.md
    references/
      plan-layout.md                  # §3 of this spec, verbatim — shared with create-master-plan
      closeout-checklist.md           # what must be complete in tasks.md and handoff.md
      rule-distillation.md            # how to read Deviations/Decisions and propose rules
      index-template.md               # INDEX.md shape
```

### 4.1 Invocation

```
/close-master-plan [<plan-folder> | <TICKET-ID>]
```

With no argument: if `<plans-root>/active/` holds exactly one folder, propose it and confirm. If it
holds several, ask which. If it is empty, fall back to the flat layout (§4.2 Step 1); if that finds
nothing either, stop and say so.

### 4.2 Workflow

**Step 1 — Locate and validate.** Resolve the project root (nearest ancestor with `.git`) and
`<plans-root>` per §3.1. Look for `<plans-root>/active/<ID>/`, then fall back to the flat
`<plans-root>/<ID>/`. **Record which layout was found — Step 8 needs it.** Require `master-plan.md`
to exist; without it this is not a plan folder and the skill stops.

**Step 2 — Git preflight.** Establish, in order:

- That this is a git repository. Every later step depends on it.
- That the working tree is clean. **If it is not, stop.** The close commit must contain the close
  and nothing else; mixing it with unfinished work destroys the property that makes every later
  step safe to inspect.
- Whether this checkout is a worktree (`git rev-parse --git-dir` differs from `--git-common-dir`)
  and which branch is checked out.
- The base branch, via `git merge-base` against the repo's default branch. If ambiguous, ask —
  `TUTORIAL.md:853` already warns that on a committed branch the base is not `HEAD`.
- The branch's commits: `git log --oneline --no-merges <base>..HEAD`. This is the raw material for
  Step 4.

Then two warnings, neither of which blocks:

- If the branch name does not reference the ticket id **and** the plan folder shows no history on
  this branch, warn that the run may be happening from the main checkout while the work lives in a
  worktree — the pack's most common mistake (`TUTORIAL.md:1030`).
- If no implementation-review artifact is present in the plan folder
  (`codex-implementation-review.md` or equivalent), say so: closing now stamps and moves a folder
  that a later review may send you back into. That review is optional (`TUTORIAL.md:76`), so
  this is information, not a gate.

**Step 3 — Establish the status.** Ask via `AskUserQuestion`: `completed`, `abandoned`, or
`superseded by <TICKET-ID>` (which prompts for the id).

`abandoned` changes the ending. The branch is about to be deleted without merging, so a close
commit made on it is discarded along with it and `closed/<ID>/` never reaches the default branch —
losing exactly the record this feature exists to preserve. The skill still performs the close here
(the folder's content lives on this branch, not on the default one), but Step 9 prints a two-part
sequence instead of one commit, and states plainly that the close is not durable until the second
part runs.

**Step 4 — Reconcile `tasks.md`.** Skipped when `tasks.md` is absent (see Step 4a).

- **Propose a phase-to-commit mapping and have the user confirm it.** Do not derive it silently.
  `tasks-template.md:81-86` requires the coordinator to replace `(pending batch)` with SHAs and to
  append a progress note saying which phases a batch covered, but it imposes **no commit-message
  format**, and `TUTORIAL.md:833` presents one-commit-per-round as what the coordinator *proposes*,
  not a contract. So: build the best mapping available from the Detailed Progress entries and the
  commit subjects, present it as a table, and let the user correct it. Rows the user cannot place
  stay `(pending batch)` rather than being guessed.
- Fill `Finished` where a phase is `completed` but the column is empty.
- Fill the `PR` column if the user supplies a number.
- Write the `Final Summary` section: what was delivered, deviations, open items, and the pointer to
  `handoff.md`. It ends with one standing caveat: **the SHAs are feature-branch commits. If this PR
  is squash-merged they will not exist on the default branch, and the PR number is the durable
  pointer.**
- Update `**Last updated:**`.

If any phase is still `pending`, `in_progress` or `blocked`, stop and ask: mark it `dropped` with a
justification line under `Decisions` (the template's own mechanism), or abort the close.

**Step 4a — No `tasks.md`.** The plan was never decomposed. Steps 4 and 5 are skipped and the
status is restricted to `abandoned` or `superseded` — a plan that was never phased out cannot be
`completed`. Step 7 stamps only the files that exist.

**Step 5 — Verify `handoff.md`.** Scan for anything the templates leave behind: any remaining
`{...}` token (`{MASTER_PLAN_FILENAME}`, `{branch-name}`, `{N}`, `{PLAN_NAME}` …) and any italic
parenthetical of the form `_(...)_` (`_(to be filled)_`, `_(none yet)_`,
`_(updates appended by …)_`). Report every one and ask whether to fill them now, together, before
continuing.

An empty *"Key deviations from the original plan"* is not an automatic failure — a plan can be
executed exactly as written — but it is confirmed out loud rather than passed over, because it is
the section reviewers spend the most attention on.

**Step 6 — Distil durable lessons into `.claude/rules/`.** Read `handoff.md`'s deviations plus
`tasks.md`'s `Decisions` and `Coordination Notes`. Propose zero or more candidate rules, following
`references/rule-distillation.md`, each as a concrete diff against the rules file matching the
stack (`.claude/rules/java.md`, etc.).

Each candidate is approved individually. **Nothing is written without an explicit yes.** Zero rules
is a valid and common outcome — a run that taught nothing durable should teach nothing durable. If
the repo has no `.claude/rules/` directory, offer to create the file; if declined, skip the step.

The test a candidate must pass: *would this still be true on the next ticket, in a different area
of the codebase?* A one-off workaround fails it. A team convention passes it.

**Step 7 — Stamp the status header** (§3.2) into `master-plan.md`, and into `tasks.md` and
`handoff.md` **if they exist**. `master-plan.md` is mandatory — Step 1 already required it, and
§3.2 makes it the file consumers read. If a header is already present, replace it rather than
adding a second.

**Step 8 — Move and index.** The source is whichever layout Step 1 recorded; the destination is
always `<plans-root>/closed/<ID>/`:

```sh
git mv docs/plans/active/GH-412 docs/plans/closed/GH-412   # active layout
git mv docs/plans/GH-412        docs/plans/closed/GH-412   # flat legacy layout
```

`git mv` rather than `mv` because it stages the move and keeps the index coherent. It does **not**
guarantee the PR renders a rename: rename display is Git's similarity detection, and this close-out
edits several of the files in the same commit that moves them. A clean rename block is likely, not
promised. If the files are untracked (a plan that was never committed), fall back to plain `mv` and
say so.

If `<plans-root>/closed/<ID>/` already exists, stop and list what is there, following the
overwrite-aware pattern of `create-master-plan` Step 1 (`SKILL.md:44-49`): the skill never silently
clobbers.

Then create or update `<plans-root>/INDEX.md` (§3.3), newest first.

**Step 9 — Report and stop.** Print a summary of what changed and the exact commands.

For `completed` and `superseded`:

```sh
git add docs/plans .claude/rules && git commit -m "GH-412: close plan"
```

For `abandoned`, the same commit **plus** the rescue that makes it durable, stated as required:

```sh
git add docs/plans .claude/rules && git commit -m "GH-412: close plan (abandoned)"
# the branch is being deleted — the archive must also land on the default branch:
git switch main && git cherry-pick <that-commit>
```

Explicit paths, never `git add -A` — `tasks-template.md`'s hard rules forbid it. Then point at
`superpowers:finishing-a-development-branch` for merging, pushing and worktree removal, which
`TUTORIAL.md:894` establishes as a separate decision. The skill does not commit, does not push,
does not switch branches, does not touch the worktree it is running inside.

### 4.3 Correcting a mistaken close

There is no reopen command in v0.1.0. A close is a commit; correcting one is `git revert` of that
commit, or a manual reverse `git mv` plus removing the header and the index row. A reopen that
looked complete would have to also reverse approved `.claude/rules/` edits, restore a header it
replaced, and undo the `tasks.md` reconciliation — reversibility the skill cannot honestly claim.
The README says this in one sentence rather than shipping a command that half-works.

## 5. Deliverable 2 — `create-master-plan` v0.4.0

### 5.1 Step 1 — folder setup

Currently `SKILL.md:42`. The plan folder becomes `<plans-root>/active/<TICKET-ID>/`, with
`<plans-root>` resolved per §3.1. If `<plans-root>/<TICKET-ID>/` exists in the flat legacy layout,
use it unchanged and note in the run summary that the project is on the old layout — the skill
never migrates on its own.

### 5.2 Step 5 — the `docs/` scan

Currently `SKILL.md:91-105`. The glob is unchanged; the reporting is partitioned:

- Matches outside `<plans-root>`, and matches under `active/` — reported exactly as today:
  relative path, matched terms, a 1–2 line excerpt.
- Matches under `closed/` — **collapsed to one line per plan folder**, no excerpt, no matter how
  many files inside it matched:
  `- GH-388 · superseded by GH-412 · closed 2026-06-02 — matched: idempotency, refund`
  The status is read from that folder's `master-plan.md` header (§3.2), which is the only file the
  scan opens beyond the matches themselves. A closed plan stays discoverable — it is a pointer the
  interview can choose to open — without spending the dossier's budget on prose that is already
  decided.

This reduces what the dossier carries, not what the scan reads: the glob and grep still traverse
every markdown file in the repo. That is by design — the cost being cut is context, which is the
scarce one.

### 5.3 New and changed references

**`references/plan-layout.md`** — §3 of this spec, shipped byte-identical in both plugins. Four
stable facts: the root resolution rule, the status vocabulary, the header format and its
authoritative carrier, the index format.

**`references/issue-specs-template.md`** — the *Related local docs* section gains a
`### Closed plans` subsection holding those one-line entries, separate from the full entries.

### 5.4 Fix the section numbering, which is wrong today

`SKILL.md:125-133` numbers the `issue.specs` sections with **Header as section 1**, making *Related
local docs* §8 and *Context Gaps* §9. `issue-specs-template.md:80-94` has no Header section and
numbers them **§7 and §8**. The two files of the same plugin disagree, in v0.3.4, before this work
touches anything — and this design's own first draft inherited the wrong one. Since 0.4.0 edits
both files anyway, align them on the template's numbering (the file that is actually written) and
renumber `SKILL.md`'s Step 6 list to match.

## 6. Deliverable 3 — the documentation surface

The new layout and the fourth step falsify prose in eight places. All of it is prose; none of it
changes behaviour.

| File | Change | Version effect |
|---|---|---|
| `plugins/create-master-plan/README.md` | Flat layout hard-coded at lines 41, 78, 111 | in 0.4.0 |
| `plugins/create-master-plan/config.md` | The knob becomes a **root** (§3.1), and documents `active/`+`closed/` | in 0.4.0 |
| `plugins/decompose-plan/**/*.md` (4 refs) | Examples → `docs/plans/active/GH-412` | patch bump |
| `plugins/plan-review/**/*.md` (4 refs) | Same | patch bump |
| `README.md` | New table row, the workflow diagram, "The three are one workflow" (line 60), **and the stale version numbers** (the table says 0.3.3/0.3.4/0.3.1; the manifests say 0.3.4/0.3.5/0.3.2) | — |
| `.claude-plugin/marketplace.json` | Fourth entry | — |
| `MANUAL.html` ×4 | **Structurally five-step**: `<h2>The five steps</h2>` (line 144), "Run all five steps" (line 605), 3 `docs/plans` refs. Becomes six, then re-synced across all four plugins | — |
| `docs/master-plan-pack/TUTORIAL.md` | 22 path refs, "Three plugins" (line 3) and §0.1 (line 86), a new "Part 6 · Close" section, a new mermaid diagram, cheat-sheet line | — |

`MANUAL.html` is currently byte-identical across the three plugins (`cdfa685f5d3c…`, 40 KB). That
invariant is worth keeping: edit once, copy to four, verify with `md5sum`. Its five-step spine is
the largest single piece of work in this deliverable and should be scoped before starting, not
discovered mid-edit.

## 7. Failure modes

| Situation | Response |
|---|---|
| Not a git repository | Stop — every step depends on git |
| Working tree dirty | **Stop.** The close commit must contain only the close |
| Run from the main checkout while work lives in a worktree | Warn — branch and plan folder do not corroborate each other |
| No implementation review present | Inform; the review is optional, so this does not block |
| A phase still `pending` / `in_progress` / `blocked` | Ask: mark `dropped` with justification, or abort |
| Phase-to-commit mapping unclear | Present the proposed table; unplaceable rows stay `(pending batch)` |
| Branch will be squash-merged | The `Final Summary` records that the SHAs are branch-local and the PR is the durable pointer |
| `handoff.md` still has `{...}` or `_(...)_` placeholders | Ask; fill them together before continuing |
| Deviations section empty | Confirm out loud, do not fail |
| `closed/<ID>/` already exists | Stop and list — never clobber |
| No `tasks.md` | Skip Steps 4–5; status restricted to `abandoned` / `superseded`; stamp only what exists |
| Plan folder in the flat legacy layout | Move from there to `closed/<ID>/`; `active/` is never required to exist |
| Plan folder untracked (never committed) | `git mv` falls back to `mv`, and says so |
| No `.claude/rules/` in the repo | Offer to create; skip if declined |
| Status `abandoned` | Step 9 prints the commit **and** the cherry-pick onto the default branch, and says the close is not durable without it |
| `closed/<ID>/master-plan.md` has no header | Consumers report the plan with status `unknown`, never skip it |
| Invoked on an already-closed plan | Report when it was closed and stop; correction is `git revert`, not a command (§4.3) |

## 8. Verification

These are markdown artifacts, so verification is a fixture and its negative cases, not a test suite.

1. **Fixture.** A throwaway git repo in the scratchpad holding `docs/plans/active/GH-999/` with
   `master-plan.md`, a `tasks.md` carrying `(pending batch)` in three rows, a `handoff.md` full of
   `_(to be filled)_` and `{branch-name}`-style tokens, and `phases/`; plus three round commits on a
   `feature/gh-999` branch. Run the skill end to end and assert each step's output.
2. **The negative cases, which matter more than the happy path**: dirty tree, a phase left
   `in_progress`, a pre-existing `closed/GH-999/`, a plan folder with no `tasks.md`, a plan in the
   flat legacy layout, and `abandoned` on a feature branch (assert the two-part command sequence).
3. **The cross-plugin contract.** After closing the fixture, run the revised Step 5 logic against a
   second ticket whose terms match `closed/GH-999/phases/PHASE-01-*.md` — a file that carries no
   header — and assert it produces exactly one collapsed line with the status read from
   `master-plan.md`. This is the finding that broke the first draft; it gets its own case.
4. **Mermaid.** Every new diagram block extracted to its own file and rendered with `mermaid-cli`
   driven by the system Chrome, with a known-bad block fed through first to prove the checker fails.
5. **Consistency sweep.** Grep for orphaned `docs/plans/GH-` references outside historical
   examples; grep for surviving "five steps" / "three plugins" wording; `md5sum` the four
   `MANUAL.html` copies.
6. **Smoke test.** `claude plugin marketplace add ./` from the local path, confirm the plugin lists
   and the skill invokes.

## 9. Decisions and rejected alternatives

| Decision | Rejected alternative | Why |
|---|---|---|
| `<plans-root>/active/` + `closed/` | `plans/archive/` outside `docs/` | Outside `docs/` would hide closed plans from the Step 5 scan entirely, but adds a second root and a second configurable path. One root keeps the config at one knob |
| | `docs/plans/_archive/` | `active`/`closed` names the state in the path and makes the move symmetric rather than a drain |
| A fourth plugin | A second skill inside `create-master-plan` | Would define the layout once, but breaks "one step, one plugin, one conversation" — the README's stated reason for the split — and drags the Jira/Linear adapters along for anyone who only wants to close |
| Teach Step 5 to rank closed plans | Leave the scan as is | Without it the layout is cosmetic: closed plans would still land in the next dossier at full weight, which is the problem being solved |
| Status read from `master-plan.md`'s header | Status read from `INDEX.md` | The header travels with the file. An index can drift; a header cannot |
| | Stamp every file in the folder | More writes, more drift, and no gain: one lookup per folder is what collapses N matches into one line |
| `INDEX.md` lists closed plans only | Also list active plans | A full board needs `create-master-plan` to write the index too — a third change and a second writer. `ls active/` answers the same question |
| Keep `INDEX.md` | Cut it (review suggestion) | With reopen cut, its cost is one append per close, and it is the only artifact in the design aimed at a human browsing rather than an agent grepping |
| Keep rule distillation | Cut it or split it out (review suggestion) | It is close-out item 3 of `TUTORIAL.md:965` and the only part of this step with durable value. Its main complication — interaction with reopen — disappeared when reopen was cut, and it is already gated on per-candidate approval with zero rules as a valid outcome |
| No reopen command | Reopen via reverse `git mv` (first draft) | It could not reverse approved rules edits, a replaced header, or the `tasks.md` reconciliation. Claiming reversibility that does not hold is worse than pointing at `git revert` |
| Propose the phase-to-commit mapping | Derive it from commit messages | No commit-message format is contracted anywhere; `TUTORIAL.md:833` calls per-round commits a default proposal. Two agents would produce different SHAs from the same branch |
| Write files, print the commands | Commit automatically | The pack's coordinator proposes commits and never runs them (`TUTORIAL.md:833`). A close-out that commits by surprise breaks that expectation on someone's real branch |
| No network | `gh` PR comment + issue close | Adds credentials, permissions and an outward-facing action to a step that is otherwise entirely local and reversible |

## 10. Review history

- **2026-08-18 · first draft.** Nine sections, reopen included, seven documentation surfaces.
- **2026-08-18 · external review** (Codex CLI, read-only, whole repo). Thirteen findings; eleven
  accepted. The consequential ones: the status-header contract could not be fulfilled because the
  scan can match unstamped files (§3.2, §5.2); the legacy flat layout contradicted the move command
  (§4.2 Step 8); phase-to-commit matching assumed a convention nothing contracts (§4.2 Step 4);
  `abandoned` lost the archive with only a warning (§4.2 Steps 3 and 9); reopen was not a genuine
  reverse (§4.3); the configurable root was ambiguous (§3.1); and the documentation inventory
  missed `create-master-plan/README.md` and the manual's five-step spine (§6). Two suggested cuts
  were declined with reasons recorded in §9. The review also surfaced a pre-existing defect
  unrelated to this work — the section-numbering disagreement now fixed in §5.4.
