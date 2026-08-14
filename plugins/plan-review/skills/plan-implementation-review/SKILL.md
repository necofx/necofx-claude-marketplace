---
name: plan-implementation-review
description: >-
  Generates a detailed, self-contained review prompt for Codex (or any external
  reviewer agent) to review the IMPLEMENTATION of a plan — whether the actual code
  changes faithfully and completely build what a planning folder specified. It works
  from two sources of truth at once: the plan (master-plan.md / PHASE-NN phase files /
  tasks.md / issue.specs) and the real git changeset (the pending-to-commit diff, or a
  diff against a base branch). Use this AFTER coding, when the user wants a
  second-opinion review that checks the code against the plan AND on its own merits,
  before committing or merging. Trigger it when the user says things like "give me a
  prompt for Codex to review the implementation", "did we build what the plan said",
  "review the pending changes against the plan", "verify the implementation of this
  ticket", "code + plan conformance review prompt", "the phases are implemented,
  generate a review prompt", or names a plan folder and asks to review the work that
  implements it — even without the word "Codex". This is the post-implementation
  sibling of plan-review-prompt (which reviews the plan BEFORE it is built); reach for
  THIS one when there is a diff to judge. Do NOT use it to perform the review yourself,
  or to review code that has no governing plan.
---

# Codex Implementation-Review Prompt Generator

## Overview

This skill turns (a) a folder of planning files and (b) the current git changeset that implements them into a single, copy-pasteable prompt that a separate Codex session (or any capable external reviewer with read access to the repo) can run to judge the implementation rigorously — on two axes at once: *did it build what the plan said*, and *is the code itself correct*.

Generating a *prompt* rather than reviewing here is the same separation of concerns the sibling relies on: the author of a change is a poor reviewer of it. A fresh session — pointed at the exact diff, the exact plan, and a sharp two-axis checklist — catches gaps the author rationalized past. The hard part, and the reason this is worth a skill, is the **grounding**: a generic "review the changes against the plan" instruction is nearly useless, because the reviewer has to rediscover which file implements which plan step, what the plan demanded exactly, and what the code looked like before. The skill does that legwork: it reads the plan, captures the real changeset, and cross-references the two into a navigational map so the reviewer starts from ground truth instead of reconstructing it.

## What this skill produces

A prompt with exactly three parts, in this order (fixed — it is what the reviewer needs and in what order):

1. **Onboarding — documents to read.** The grouped, verified reading list: the plan + spec (what was *promised*); the changeset (what *landed*) with a navigational changed-file → plan-item map and the exact commands to reproduce the diff; the repo surface to read in full so the reviewer can judge correctness; and the repo's own coding-standard / rules files.
2. **Review instructions.** A two-dimensional checklist — Dimension 1 plan-conformance, Dimension 2 code quality & technical correctness.
3. **Output format.** Verdict, conformance matrix, findings table, status-honesty check, deviation rulings, what's-correct.

The prompt is always self-contained and read-only: it names the repo root, the base ref the diff is taken against, the reading list, the instructions, the format, and the reviewer constraints (read-only, regenerate the diff live, cite `file:line`, separate verified facts from opinions).

## Workflow

### Step 1 — Resolve inputs

Take whatever the user already gave you in the invocation; ask only for what's missing.

- **Planning folder** — the directory holding the plan files. If not provided, ask for it (absolute path, or relative to the repo). Do not guess.
- **Diff base** — what the changeset is measured against. Default to `HEAD` (the literal "pending to commit" changes: staged + unstaged + untracked). But some workflows commit before review (e.g., a repo where the assistant cannot commit and the developer commits by hand, so the work is already on a feature branch). So also support a base **branch/ref** (`master`, `main`, the merge-base) to review an already-committed branch. If the user named one ("against master"), use it. Otherwise default to `HEAD` and confirm in Step 3 that a diff actually exists — if the working tree is clean, fall back to asking whether to diff against a base branch instead.

### Step 2 — Detect the plan files and the repo root

List the folder (top level; and `phases/` if present). Identify, tolerantly of variants and casing:

- **Master plan**: `master-plan.md` / `MASTER_PLAN.md` / `plan.md` / `PLAN.md`, or the single `*.md` if exactly one exists.
- **Spec / ticket (source of truth)**: `issue.specs` / `*.spec` / `*.specs` / `SPEC.md` / an `analysis.md`.
- **Decomposition + tracking**: `phases/PHASE-*.md`, `tasks.md`, `execute-plan.md`, `handoff.md`.

Resolve the **repo root**: walk up from the folder to the nearest ancestor containing a `.git` entry. State it as an absolute path; use **repo-relative paths** everywhere else. If there is no `.git` ancestor, use the folder's top-most sensible parent and say so.

There is no master/phases/all mode here (unlike the sibling): the implementation is judged against the **whole** plan that exists — master plan + phases + spec + tracker. Read all the plan artifacts present. Report what you found before continuing, so the user can correct a misdetection.

### Step 3 — Capture the changeset (the new core)

Using **read-only git only**, from the repo root:

1. `git status --porcelain` — the full list of staged, unstaged, and untracked paths.
2. The tracked modifications:
   - base = `HEAD` (pending-to-commit): `git diff HEAD --stat`, then `git diff HEAD`.
   - base = a branch/ref (already-committed work): usually `git diff <base>...HEAD --stat` / `git diff <base>...HEAD` (three-dot = changes since the merge-base, i.e. "what this branch added"). Choose two-dot vs three-dot to match the user's intent and **state which you used**.
3. **Untracked files** (`??` in porcelain) carry no diff — list them; the reviewer must read each in full. This is the easiest thing to drop; don't.

**Guard the empty case.** If the changeset is empty or trivially small (the work was already committed past `HEAD`, or nothing is staged/changed), STOP and resolve it with the user before emitting a prompt — a review prompt over an empty diff is vacuous. Offer the likely fix (diff against the base branch instead of `HEAD`).

Capture the changed-file list and per-file line counts; you fold them into the onboarding as a navigational map and a drift tripwire.

### Step 4 — Build the onboarding (the core; do this carefully)

Read the plan files and the changeset, and assemble a grounded, **verified** reading list. Group into these categories (omit any that are empty); annotate every entry with a one-line "why this matters for the review":

**A. The plan — what was promised.** The master plan; the spec/ticket (source of truth for requirements & acceptance criteria); the phase files; and the tracker (`tasks.md`) *with its claimed phase statuses*.

**B. The changeset — what landed.**
- The exact commands to **reproduce the diff live** (`git status --porcelain`; `git diff <base>` in the form you chose; read each untracked file). Tell the reviewer to regenerate, never trust a paste.
- A **navigational changed-file → plan-item map**: for each changed path, its line delta and which plan unit (finding / requirement / phase + step) appears to govern it. This map is **navigational, not evaluative** — it tells the reviewer *where to look*, NOT whether the implementation is correct or complete. The Implemented / Partial / Missing verdict is the reviewer's job (Dimension 1), never a value the skill pre-fills; pre-judging anchors the reviewer to the author's conclusion and kills the fresh-eyes value that is the whole point.
- A `git status --porcelain` + `--stat` **snapshot**, framed as a drift tripwire: if the reviewer's live `git diff <base> --stat` differs from it, the tree changed since this prompt was written — review the live state and note the drift.

**C. Repo surface to read in full.** A diff hunk is not enough to judge correctness — the surrounding code, the callers of a changed function, the tests, and any touched config/doc must be read whole. List the post-change files the reviewer should open completely: the changed files themselves; the callers / blast-radius of modified shared code; the test files; the touched configs. Annotate with what to check.

**C-bis. CodeGraph, when the repo has one.** If a `.codegraph/` directory exists at the repository root, add a short **Tooling** block to the generated prompt telling the reviewer to reach for it instead of a search loop: `codegraph explore "<question>"` returns the relevant symbols' verbatim line-numbered source plus the call paths between them in one call; `codegraph impact <symbol>` gives the blast radius of a changed shared symbol — the group-C question, answered against the real graph instead of a text match, including dynamic dispatch a `grep` cannot follow; `codegraph affected <changed files>` names the tests the changeset reaches, which is how the reviewer judges whether the tests that landed actually cover it. Two constraints go in the block: fall back to ordinary search if the command is unavailable in the reviewer's environment, and never index the repository — indexing is the user's decision. If there is no `.codegraph/` directory, omit the block entirely rather than mentioning a tool the reviewer cannot use.

**D. The repo's own standards — discover, don't assume.** Find and name the standards the code must satisfy: root `AGENTS.md` / `AGENTS.md` / `GEMINI.md`, `.Codex/rules/*`, `CONTRIBUTING` / coding-standards docs, `.editorconfig`, linter/formatter configs (eslint, ruff, stylecop, etc.). Do **not** hardcode any specific rule — point the reviewer at the files so it checks the code against the project's *actual* conventions. (Same discover-and-verify move the sibling does for paths.)

**E. Optional calibration (read if accessible).** A predecessor review, the master-plan-review prompt if one was generated for this folder, canonical examples of the same kind of change elsewhere in the repo.

**Verify every cited path exists** (Glob / ls). Keep the real ones. For a path the plan cites that does NOT exist, keep it but flag it `(referenced by the plan but NOT found — confirm whether it is a stale reference)` — a dangling reference is itself a finding.

### Step 5 — Assemble the prompt

Read `references/implementation-review.md` and embed its **Review instructions** and **Output format** blocks, tailoring the `<...>` placeholders to what you discovered: the real stack, the real build/test commands, the base ref, and 0–3 "Scrutinise especially" bullets naming the highest-risk changes (a security-sensitive edit, a shared-code change with wide blast radius, a keystone the rest depends on).

Prompt skeleton:

```
You are a senior <role> reviewing the IMPLEMENTATION of a plan. Judge two
things: did the changeset build what the plan specified, and is the code itself
correct. Read-only; regenerate the diff live; cite file:line (or the diff hunk)
for every claim.

# Repo & changeset
Run from the repo root: <abs repo root>. <one line on the stack / platform>.
Changeset under review = <base ref + the exact git command>. Reproduce it live
before reviewing.

# What you're reviewing
<one paragraph: the plan, the changeset, and the two-axis standard to hold it to>

# Tooling  (only if the repo is indexed — omit this section otherwise)
<the codegraph note>

# Onboarding — read these first (in order)
## A. The plan — what was promised
## B. The changeset — what landed  (navigational map; the verdicts are yours, not pre-filled)
## C. Repo surface to read in full
## D. The repo's own standards
## E. Optional calibration (read if accessible)

# How to perform the review
<the two-dimensional checklist from the reference>

# Output format
<the output format from the reference>
```

### Step 6 — Deliver

Print the finished prompt inline in a single fenced ```` ```markdown ```` block so the user can copy it in one go. Then offer to save it to a file for piping into Codex — default suggestion `<folder>/codex-implementation-review-prompt.md` (or `C:\tmp\` / `/tmp` if the folder should stay clean). Write the file only if the user agrees.

Do not review the changeset yourself — the skill's deliverable is the prompt, and the whole value comes from a reviewer with no memory of authoring the change. (If the user instead wants *you* to review it, that's a different request; point them at their `code-review` tooling.)

### Step 7 — Optionally run the review

Running the prompt is **optional and never automatic**. Once it is saved to a file, offer to run it, and run it only if the user agrees:

```sh
codex exec --sandbox read-only ${CODEX_OPENROUTER_MODEL:+-m "$CODEX_OPENROUTER_MODEL"} \
  -o <folder>/codex-implementation-review.md \
  < <folder>/codex-implementation-review-prompt.md
```

- Run it **from the repo root** — the prompt's paths are repo-relative and it regenerates the diff itself.
- **Take the model from the environment, never hardcode one.** If `CODEX_OPENROUTER_MODEL` is set, pass its value as `-m <slug>`; if it is unset, omit `-m` and let the CLI use the model from its own config. The expansion above does exactly that in one line.
- `--sandbox read-only` is deliberate and sufficient: the reviewer reads the repo and runs read-only git; it must not touch the tree it is judging.
- `-o <file>` captures the reviewer's final report verbatim, so the user reads the review itself rather than a retelling of it.
- Expect minutes, not seconds, on a large changeset. Run it in the background if the harness supports that, rather than blocking on a foreground timeout.
- **Don't let the tree drift during the run.** The reviewer reproduces the diff live; edits made while it works produce findings against code that no longer exists.
- If `codex` is not installed, or not authenticated (`codex login`), say so and stop. The prompt file is still the deliverable — any capable reviewer with read access to the repo can take it; Codex is only who it was written for.

When the run finishes, point the user at the report file. **Do not summarize it in place of the file**, and do not start applying findings unprompted: they are claims to verify against the code, and accepting a wrong one costs more than missing a marginal one. Invoking a separate reviewer is not the same as reviewing — you still do not review the changeset yourself.

## References

- `references/implementation-review.md` — the embeddable **Review-instructions** (two-dimensional: plan-conformance + code-quality/technical-correctness) and **Output-format** blocks. Read it in Step 5 and embed the two fenced blocks, lightly tailored to the stack and the highest-risk changes you found.

## Principles

- **Two sources of truth, both verified.** The plan says what *should* exist; the diff shows what *does*. The entire value is judging one against the other from ground truth — so the onboarding must point at both precisely, and the reviewer must read the post-change code, not just the hunks.
- **Navigational, not evaluative.** The skill hands the reviewer a map (which file implements which plan item), never a verdict. Pre-filling "this is correctly implemented" anchors the reviewer to the author's conclusion and destroys the fresh-eyes value. Verdicts are the reviewer's output.
- **Regenerate, never trust a paste.** Working trees drift. The prompt makes the reviewer reproduce the diff live and treats the embedded snapshot only as a drift tripwire.
- **Discover the project's standards; don't impose generic ones.** Point the reviewer at the repo's real rules files. The same skill must serve a Delphi, Python, or TypeScript repo, each with its own conventions — so detect the stack and standards, don't hardcode them.
- **Scope the reviewer to this changeset against this plan.** Flag where the code contradicts the plan or the spec, and where the code is wrong on its own merits; but don't re-litigate the plan's settled product decisions (that is the sibling's job, pre-build).
- **Demand evidence.** Every generated prompt instructs the reviewer to cite `file:line` (or the diff hunk), separate "the plan said X; the code at `file:line` does X" (conformance facts) from "`file:line` has bug Y" (quality findings) from "I'd prefer Z" (opinions), and say "I couldn't verify X" rather than assume. This is what makes the resulting review trustworthy.
