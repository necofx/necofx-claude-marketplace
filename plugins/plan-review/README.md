# plan-review

Two skills that generate a **review prompt** for a fresh external reviewer — one for the plan before it is built, one for the code against the plan afterwards. Neither performs the review itself, and that is deliberate: the author of a plan is a poor reviewer of it.

| Skill | Reviews | Reach for it |
|---|---|---|
| `plan-review-prompt` | The plan, before anything is built — a master plan, a decomposed set of phase files, or both | After [`create-master-plan`](../create-master-plan/) or [`decompose-plan`](../decompose-plan/), before you start executing |
| `plan-implementation-review` | The real changeset against the plan, afterwards | After the phases are implemented, before committing or merging |

## Credits

**The workflow is not ours.** It was designed by someone else, who shared their files directly, and it is published here with their permission. Both skills in this plugin are their files, unchanged — unlike the other two plugins in this workflow, nothing here needed adapting, because a review prompt does not care which tracker the plan came from.

`MANUAL.html` in this folder is their manual for the complete five-step workflow; these two skills sit at steps 4–5.

---

## Setup

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install plan-review@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH — pass `https://github.com/necofx/necofx-claude-marketplace.git` instead.

Then start a new conversation — skills load at session start.

**That is the whole setup.** No configuration, no placeholders, no required plugins. Both skills read the plan folder and the repository with ordinary tools, and `plan-implementation-review` uses read-only git (`status`, `diff`) and nothing else.

### What you need on the other side

The generated prompt is self-contained and works with any capable reviewer that has read access to the repository. It was written for **Codex**; a second Claude Code session, another agent, or a human reviewer all work. The only real requirement is that the reviewer can open the repository and run `git diff` itself — the prompt tells it to regenerate the diff live rather than trust a paste.

---

## Tutorial A: review the plan before you build it

Run this after decomposing and before you paste the coordinator prompt. Catching a wrong phase boundary here costs minutes; catching it in round 2 costs a round and a confused human.

### 1. Generate the prompt

```
/plan-review-prompt
```

It asks two things — answer them in the invocation if you prefer:

- **Which planning folder?** Absolute or repo-relative. It does not guess.
- **What are we reviewing?** `master plan`, `phases`, or `all`. If the folder has no `phases/` yet, only "master plan" is meaningful and it says so rather than asking a pointless question.

It reports what it detected — which file is the master plan, whether phases exist, how many — before continuing, so a misdetection is visible and correctable rather than silent.

### 2. What it actually does, and why it takes a moment

The checklist is the easy half. **The onboarding is what makes the review worth reading.** A generic "read the docs and review the plan" instruction is nearly useless, because the reviewer cannot verify a plan it has no surface area for.

So the skill reads the plan files, harvests every path they cite — from the explicit sections first ("Documents to Read", "Files", "Related Docs", each phase's Dependencies) and then from the prose — and **verifies each of those paths actually exists**. Real ones go into a grouped, annotated reading list. A path the plan cites that does *not* exist is kept and flagged as a possible stale reference, because a dangling reference is itself a finding.

The result is four groups: the plan under review, the source of truth it must honour, the repo surface needed to check its `file:line` claims, and optional calibration material.

### 3. Run it and read the result

The prompt is printed inline in a single fenced block so you can copy it in one go. It then offers to save it to `<folder>/codex-review-prompt-<mode>.md` — it only writes the file if you say yes.

Paste it into a fresh Codex session (or whatever reviewer you use) from the repo root.

A master-plan review checks fidelity to ground truth, technical correctness, soundness of the key decisions, completeness and internal consistency. A decomposition review checks coverage and traceability, round and dependency correctness, the file-conflict matrix, phase self-containment, `file:line` accuracy and right-sizing.

---

## Tutorial B: review the implementation against the plan

Run this after the last round, before committing or merging.

### 1. Generate the prompt

```
/plan-implementation-review
```

It asks:

- **Which planning folder?** Same as above.
- **What is the diff measured against?** Default `HEAD` — the literal pending-to-commit changes, staged plus unstaged plus untracked. Say `master` (or any branch/ref) instead if the work is already committed on a feature branch, which is the normal case when your agents cannot commit and you commit by hand.

**If the working tree is clean it stops and asks**, rather than emitting a prompt over an empty diff. A review of nothing is vacuous, and the usual fix is that you meant to diff against a base branch.

### 2. What it builds

Two sources of truth at once — the plan (what was *promised*) and the changeset (what *landed*) — plus the cross-reference between them:

- The exact commands to **reproduce the diff live**, because working trees drift. The embedded `--stat` snapshot is framed only as a tripwire: if the reviewer's live diff differs from it, the tree changed since the prompt was written.
- **Untracked files listed separately.** They carry no diff and the reviewer must read each in full. This is the easiest thing to drop and the skill does not drop it.
- A **changed-file → plan-item map**, which is navigational and never evaluative. It tells the reviewer *where to look*, never whether the code is right. Pre-filling an "implemented / partial / missing" verdict would anchor the reviewer to your conclusion and destroy the fresh-eyes value that is the entire point.
- **Your repository's own standards, discovered rather than assumed** — `CONTRIBUTING`, coding-standards docs, `.editorconfig`, linter and formatter configs. The reviewer is pointed at them so it judges your code against your conventions, not generic ones.

The review then runs on two axes: did the changeset build what the plan specified, and is the code correct on its own merits.

### 3. Read the findings like findings

**They are claims, not verdicts.** Some are wrong. Verify each against the code before changing anything — accepting a wrong finding costs more than missing a marginal one. The generated prompt makes the reviewer cite `file:line`, separate conformance facts from quality findings from preferences, and say "I couldn't verify X" rather than assume, which is exactly what makes them checkable.

---

## How this relates to `/code-review`

They answer different questions and do not replace each other.

- **`/code-review`** reviews the branch diff on its own merits, inside Claude Code, and can apply its findings with `--fix` or post them to a PR with `--comment`.
- **`plan-implementation-review`** asks whether the changeset built *what the plan specified*, from a separate session with fresh eyes. Plan conformance is not something a diff review can judge, because judging it requires the plan.

`MANUAL.html` runs `/code-review` as step 4 and reaches for these prompts when a genuine second opinion is wanted.

## Limits

- **They generate a prompt; you run the review.** By design, and it is the whole separation-of-concerns argument.
- **Artifact detection is convention-based.** It looks for `master-plan.md` / `MASTER_PLAN.md` / `plan.md`, `issue.specs`, `phases/PHASE-*.md`, `tasks.md`, `execute-plan.md` and `handoff.md`. Both skills report what they found before continuing.
- **`plan-review-prompt` reviews plans, not arbitrary code.** For source with no governing plan behind it, use `/code-review`.
