---
name: plan-review-prompt
description: >-
  Generates a detailed, self-contained review prompt for Codex (or any external
  reviewer agent) to review an implementation plan — a master plan, a decomposed
  set of phase files, or both. Use this whenever the user wants an external or
  second-opinion review of a planning folder: a PRP, a master-plan.md, decomposed
  PHASE-NN phase files, tasks.md / execute-plan.md / handoff.md. Trigger it when
  the user says things like "give me a prompt for Codex to review my plan",
  "I want Codex to review the master plan / the phases", "prompt to review this
  planning folder", "hand this plan to another agent for review", or names a
  planning folder and asks for a review prompt — even if they don't say the word
  "Codex". The skill's value is that it discovers the REAL docs, code, scripts,
  and configs the plan references (and verifies the paths exist), then assembles
  an onboarding section, review instructions, and an output format. Do NOT use it
  to actually perform the review yourself, or to review arbitrary source code that
  is not an implementation plan.
---

# Codex Plan Review Prompt Generator

## Overview

This skill turns a folder of planning files into a single, copy-pasteable prompt that a separate Codex session (or any capable external reviewer with read access to the repo) can run to review the plan rigorously.

The point of generating a *prompt* rather than doing the review here is separation of concerns: the author of a plan is a poor reviewer of it. A fresh Codex session, pointed at the exact files and given a sharp checklist, catches gaps the author rationalized past. The hard part — and the reason this is worth a skill — is the **onboarding**: a generic "read the docs and review the plan" instruction is nearly useless, because the reviewer can't verify a plan it doesn't have the surface area for. The skill's job is to read the plan, extract the *specific* docs, source files, scripts, and configs it depends on, verify those paths are real, and hand the reviewer a precise reading list so it can check the plan against ground truth.

## What this skill produces

A prompt with exactly three parts, in this order (this structure is fixed — it is what the reviewer needs and in what order):

1. **Onboarding — documents to read.** The grouped, verified list of files the reviewer must read before forming an opinion: the plan files themselves, the source-of-truth spec, the repo surface (code/scripts/configs/tests) the plan touches so file:line references can be checked, and optional calibration material.
2. **Review instructions.** A mode-specific checklist of what to verify and how (different failure modes for a master plan vs. a decomposition).
3. **Output format.** The exact shape the reviewer's findings should take (verdict, findings table, answers to high-judgment questions, etc.).

The prompt is always self-contained: it names the repo root, what is being reviewed, the reading list, the instructions, the format, and the reviewer constraints (read-only, cite `file:line`, separate verified facts from opinions).

## Workflow

### Step 1 — Resolve inputs

You need two things. Take whatever the user already gave you in the invocation; ask only for what's missing.

- **Planning folder** — the directory holding the plan files. If not provided, ask for it (absolute path, or relative to the repo). Do not guess.
- **Review target** — one of: **master plan**, **phases (decomposed plan)**, or **all**. If the user didn't say, ask. If the folder has no decomposed phase files (see Step 2), only "master plan" is meaningful — say so and proceed with that rather than asking a pointless question.

### Step 2 — Detect the planning files and the repo root

List the folder (top level; and `phases/` if present). Identify, by these conventions (be tolerant of variants and casing):

- **Master plan**: `master-plan.md` / `MASTER_PLAN.md` / `plan.md` / `PLAN.md`, or the single `*.md` if exactly one exists.
- **Spec / ticket (source of truth)**: `issue.specs` / `*.spec` / `*.specs` / `SPEC.md` / `issue.spects` (a known typo) / an `analysis.md`.
- **Decomposition artifacts** (for "phases" / "all"): `phases/PHASE-*.md`, `tasks.md`, `execute-plan.md`, `handoff.md` / `HANDOFF.md`.

Also resolve the **repo root**: walk up from the folder to the nearest ancestor directory containing a `.git` entry. The generated prompt states the repo root as an absolute path and then uses **repo-relative paths** everywhere else (cleaner, and matches how a reviewer runs from the repo root). If there is no `.git` ancestor, use the folder's top-most sensible parent and say so.

Report what you found (which plan file, whether phases exist, how many) before continuing, so the user can correct a misdetection.

### Step 3 — Build the onboarding (the core; do this carefully)

This is where the skill earns its keep. Read the relevant plan files for the chosen mode and extract the **specific** files the reviewer must read. Do not write a generic list.

1. **Read the plan files** for the mode:
   - master plan → the master plan file + the spec.
   - phases → every `phases/PHASE-*.md` + `tasks.md` + `execute-plan.md` + `handoff.md` + the master plan (as the source of truth the phases must cover).
   - all → the union.
2. **Extract referenced paths.** Harvest from the explicit sections first ("Documents to Read", "Related Docs", "Files", "Pre-flight Checklist", "Tech Stack", "Technical Requirements", each phase's "Files" + "Documents to Read" + "Dependencies"), then sweep the prose for path-like tokens — anything matching a real directory prefix (`docs/`, `src/`, `scripts/`, `helm/`, `.gitlab/`, `tests/`, `app/`, `lib/`, …) or a file extension (`.md .cs .ts .tsx .js .py .sh .ps1 .yml .yaml .json .csproj .sln .pas .dproj`, `Dockerfile`, `Makefile`, …). A quick `grep`/Grep for `[\w./-]+\.\w+` and for the directory prefixes is a good net.
3. **Verify every path exists** (Glob / ls). Keep the real ones. Drop obviously-generic mentions; for a path the plan cites that does NOT exist, keep it but flag it `(referenced by the plan but NOT found — the reviewer should confirm whether it is a stale reference)` — a dangling reference is itself a finding worth surfacing.
4. **De-duplicate and group** into these categories (omit any that are empty):
   - **A. The plan under review** — the plan/phase/orchestration files themselves.
   - **B. Source of truth** — the spec/ticket the plan must faithfully honor.
   - **C. Repo surface to verify** — the code, scripts, configs, and tests the plan touches, so the reviewer can confirm the plan's `file:line` references resolve and its claims about the code are true. This is the most important group and is usually the longest. **When `.codegraph/` exists, build this group from the graph**: `codegraph explore "<the area this plan touches>"` returns the real files and their dependents in one call, and `codegraph impact <symbol>` surfaces the caller the plan never mentioned — which is the same omission that later turns two "parallel" phases into a file conflict. Harvesting only what the plan cites reproduces the plan's own blind spots. Fall back to ordinary search when the repo is not indexed, and never index it yourself.
   - **D. Optional calibration** — predecessor plans, canonical examples in the same repo, or the skill/templates the plan was generated from, if discoverable. Mark as "read if accessible".
5. **Annotate each entry** with a one-line "why this matters for the review". A bare path list is weak; the reason is what directs the reviewer's attention.
6. **CodeGraph, when the repo has one.** If a `.codegraph/` directory exists at the repository root, add a short **Tooling** block to the generated prompt: tell the reviewer to prefer `codegraph explore "<question>"` over `Glob`/`Grep`/`Read` loops when checking the plan's claims about the code, because one call returns the relevant symbols' verbatim line-numbered source, the call paths between them and what depends on them — and it follows dynamic dispatch (DI resolution, registries, callbacks) that a text search cannot, which is exactly where a plan's "this is isolated" claim tends to be wrong. `codegraph impact <symbol>` answers "what does changing this reach". Two constraints go in the block: fall back to ordinary search if the command is unavailable in the reviewer's environment, and never index the repository — indexing is the user's decision. If there is no `.codegraph/` directory, omit the block entirely rather than mentioning a tool the reviewer cannot use.

When the mode is **all**, build ONE merged onboarding (the plan artifacts of both kinds, the shared source of truth once, and the union of the repo surface) — do not duplicate the shared files across two lists.

### Step 4 — Assemble the prompt

Compose the final prompt in this order. Read the matching reference file(s) and embed their **Review instructions** and **Output format** blocks, lightly tailored to what you discovered (substitute the real stack, build/test commands, and any plan-specific high-risk areas you noticed):

- **`references/master-plan-review.md`** — for "master plan".
- **`references/decomposed-plan-review.md`** — for "phases".
- For **"all"** — embed BOTH instruction sets under one prompt: a shared header + the merged onboarding, then a "Part 1 — Master plan review" section (its instructions) and a "Part 2 — Decomposition review" section (its instructions), then a single combined output format that asks for both verdicts. The references tell you how to merge.

Prompt skeleton:

```
You are a senior <role> reviewing an implementation plan. Review the PLAN, not <out-of-scope>. This is read-only; cite file:line for every claim.

# Repo
Run from the repo root: <abs repo root>. <one line on the stack / platform>.

# What you're reviewing
<one paragraph: what this plan is, the mode, and the standard to hold it to>

# Tooling  (only if the repo is indexed — omit this section otherwise)
<the codegraph note>

# Onboarding — read these first (in order)
## A. The plan under review
- <path> — <why>
## B. Source of truth
- <path> — <why>
## C. Repo surface to verify the plan's file:line references
- <path> — <why / what to check>
## D. Optional calibration (read if accessible)
- <path> — <why>

# How to perform the review
<the mode-specific checklist from the reference>

# Output format
<the mode-specific output format from the reference>
```

### Step 5 — Deliver

Print the finished prompt inline in a single fenced ```` ```markdown ```` block so the user can copy it in one go. Then offer to save it to a file for piping into Codex — default suggestion `<folder>/codex-review-prompt-<mode>.md` (or `C:\tmp\` / `/tmp` if the folder should stay clean). Only write the file if the user agrees.

Do not review the plan yourself — the skill's deliverable is the prompt, and the whole value comes from a reviewer with no memory of authoring the plan. (If the user instead wants *you* to review it, that's a different request; point them at their `code-review` tooling.)

### Step 6 — Optionally run the review

Running the prompt is **optional and never automatic**. Once it is saved to a file, offer to run it, and run it only if the user agrees:

```sh
codex exec --sandbox read-only ${CODEX_OPENROUTER_MODEL:+-m "$CODEX_OPENROUTER_MODEL"} \
  -o <folder>/codex-review-<mode>.md \
  < <folder>/codex-review-prompt-<mode>.md
```

- Run it **from the repo root** — the prompt's paths are repo-relative.
- **Take the model from the environment, never hardcode one.** If `CODEX_OPENROUTER_MODEL` is set, pass its value as `-m <slug>`; if it is unset, omit `-m` and let the CLI use the model from its own config. The expansion above does exactly that in one line.
- `--sandbox read-only` is deliberate: the reviewer reads the repo and must not write to it.
- `-o <file>` captures the reviewer's final report verbatim, so the user reads the review itself rather than a retelling of it.
- Expect minutes, not seconds, on a large plan. Run it in the background if the harness supports that, rather than blocking on a foreground timeout.
- If `codex` is not installed, or not authenticated (`codex login`), say so and stop. The prompt file is still the deliverable — any capable reviewer with read access to the repo can take it; Codex is only who it was written for.

When the run finishes, point the user at the report file. **Do not summarize it in place of the file**, and do not act on its findings unprompted: they are claims to verify, not instructions. Invoking a separate reviewer is not the same as reviewing — you still do not review the plan yourself.

## References

- `references/master-plan-review.md` — the Review-instructions + Output-format blocks for reviewing a **master plan** (fidelity to ground truth, technical correctness, key-decision soundness, completeness, internal consistency). Read it in Step 4 when the mode is "master" or "all".
- `references/decomposed-plan-review.md` — the Review-instructions + Output-format blocks for reviewing a **decomposition** (coverage/traceability, round & dependency correctness, file-conflict-matrix verification, phase self-containment, file:line accuracy, right-sizing, executability risks). Read it in Step 4 when the mode is "phases" or "all".

## Principles

- **The onboarding must be real and verified, never generic.** The entire value proposition is that the reviewer is pointed at the exact surface area the plan touches. A made-up or unverified reading list is worse than none.
- **Scope the reviewer to the plan, not the world.** A master-plan review judges the plan's correctness and completeness — not the underlying product decisions (which were made elsewhere). A decomposition review judges the breakdown — not the master plan's technical calls. Say so in the prompt; tell the reviewer to flag contradictions with the source of truth but not re-litigate settled decisions.
- **Demand evidence.** Every generated prompt instructs the reviewer to cite `file:line`, separate verified facts from opinions, and say "I couldn't verify X" rather than assume. This is what makes the resulting review trustworthy.
- **Stay tech-stack agnostic.** Detect the stack from the plan (its build/test commands, file extensions, rules files) and substitute the real commands into the prompt. Don't hardcode .NET/Helm/React assumptions — the same skill should serve a Delphi plan or a Python plan.
