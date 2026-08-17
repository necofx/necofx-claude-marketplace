# plan-review

Two skills that generate a **review prompt** for a fresh external reviewer — one for the plan before it is built, one for the code against the plan afterwards. Neither performs the review itself, and that is deliberate: the author of a plan is a poor reviewer of it. Both will hand the prompt to the Codex CLI for you if you want, so "external" costs you nothing but the wait.

| Skill | Reviews | Reach for it |
|---|---|---|
| `plan-review-prompt` | The plan, before anything is built — a master plan, a decomposed set of phase files, or both | After [`create-master-plan`](../create-master-plan/) or [`decompose-plan`](../decompose-plan/), before you start executing |
| `plan-implementation-review` | The real changeset against the plan, afterwards | After the phases are implemented, before committing or merging |

## Is it optional?

**Yes — entirely.** The workflow closes without either skill: plan, decompose, execute, and `/code-review` inside Claude Code covers the ordinary review. Nothing downstream waits on a peer review, and no other plugin reads its output.

Reach for it when you want a genuine second opinion — a reviewer that never saw the plan being written, running in its own session, with its own context and its own model. That is the one thing `/code-review` structurally cannot give you, because it is the same session that did the work.

**The reviewer does not have to be Codex.** The generated prompt is self-contained and works with anything that can read the repository: a second Claude Code session, another agent, or a person. Codex is simply who it was written for, and the only one this plugin can run for you.

## Credits

**The workflow is not ours.** It was designed by someone else, who shared their files directly, and it is published here with their permission. Both skills are their files with one addition of ours: an optional final step that runs the generated prompt through the Codex CLI, rather than leaving you to run it by hand. Nothing else needed adapting — unlike the other two plugins in this workflow, a review prompt does not care which tracker the plan came from.

`MANUAL.html` in this folder is their manual for the complete five-step workflow; these two skills sit at steps 4–5.

---

## Setup

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install plan-review@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH — pass `https://github.com/necofx/necofx-claude-marketplace.git` instead.

Then start a new conversation — skills load at session start.

**That is the whole setup.** No configuration, no placeholders, no required plugins. Both skills read the plan folder and the repository with ordinary tools, and `plan-implementation-review` builds its prompt from read-only git (`status`, `diff`). The only thing either one does beyond reading is launch Codex — and only if you ask it to.

### What you need on the other side

The generated prompt is self-contained and works with any capable reviewer that has read access to the repository. It was written for **Codex**; a second Claude Code session, another agent, or a human reviewer all work. The only real requirement is that the reviewer can open the repository and run `git diff` itself — the prompt tells it to regenerate the diff live rather than trust a paste.

If you want the skills to run the review for you, you need the [Codex CLI](https://github.com/openai/codex) on your `PATH` and logged in:

```sh
codex --version
codex login          # only if you aren't already
```

Neither skill needs it to generate a prompt. Without Codex installed, everything still works — you just paste the prompt somewhere yourself.

Optionally, export `CODEX_MODEL` with the name of the model you want the reviewer to run on. Unset, the run uses Codex's own default — no special provider or routing.

### If the repo is indexed by CodeGraph, the prompt says so

When a `.codegraph/` directory exists at the repo root, CodeGraph is used on **both sides of the handoff**.

The skills use it while building the prompt: the reading list of callers, blast radius and affected tests is resolved with `codegraph impact` and `codegraph affected` instead of being inferred from the diff or from what the plan happens to cite. A list built the other way inherits the blind spots of the thing under review, which defeats the point.

And the prompt carries a **Tooling** block telling the reviewer to answer "who calls this, what does this change reach, which tests cover it" the same way rather than with a `grep`/read loop. One call returns the symbols' line-numbered source plus the call paths between them, and it follows dynamic dispatch — registries, DI resolution, callbacks — that a text search cannot. That is precisely where "this change is isolated" turns out to be false.

It works inside `--sandbox read-only`; that was verified, not assumed. One caveat behind the verification: the index's daemon was already running at the time, so if the CLI ever has to start one it may want write access. The block therefore tells the reviewer to fall back to ordinary search if the command is not available to it.

No `.codegraph/` directory means no block at all — the prompt never mentions a tool the reviewer cannot use. And neither skill will index your repository: that is your decision, and it writes hundreds of megabytes. The [`decompose-plan` README](../decompose-plan/README.md#4-optional-codegraph) has the install if you want one.

### How the review is run

Both tutorials below end the same way, so the mechanics are here once.

The skill offers to run the review after it saves the prompt. Say yes and it executes exactly this, from the repo root:

```sh
codex exec --sandbox read-only ${CODEX_MODEL:+-m "$CODEX_MODEL"} \
  -o <report>.md < <prompt>.md
```

Say no — or run it yourself later, or on another machine — and the same line is yours to paste. Four things about it are deliberate:

- **`--sandbox read-only`** is enough, and is the point. The reviewer reads the repository and runs read-only git; it must never touch the tree it is judging. A review that edits your code is not a review.
- **`-o <report>.md`** captures the reviewer's final report to a file, verbatim. You read the review itself, not a retelling of it — which is also why the skill points you at the file instead of summarizing it back at you.
- **The model comes from the environment.** Set `CODEX_MODEL` to a name and the run uses it; leave it unset and the `${...:+...}` expansion collapses to nothing, so the CLI falls back to Codex's own default — no special provider or routing. Nothing here hardcodes a model, which matters because a reviewer is exactly the place you want to reach for a different one than the one that wrote the code.
- **It takes minutes**, not seconds, on a real changeset, so the skill backgrounds it where the harness allows. Don't edit the tree while it works: the reviewer regenerates the diff live, and edits mid-run produce findings against code that no longer exists.

Prefer a different reviewer? The prompt is plain markdown. Paste it into a fresh session of whatever you use, from the repo root.

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

### 3. Run the review

The prompt is printed inline in a single fenced block so you can copy it in one go. It then offers to save it to `<folder>/codex-review-prompt-<mode>.md` — it only writes the file if you say yes — and, once saved, offers to run it.

Say yes to both and you never leave the session. The command it runs, and the one to paste if you'd rather drive it yourself:

```sh
codex exec --sandbox read-only ${CODEX_MODEL:+-m "$CODEX_MODEL"} \
  -o docs/plans/GH-412/codex-review-all.md \
  < docs/plans/GH-412/codex-review-prompt-all.md
```

Both offers are opt-in: decline either one and you still have the prompt, which is the deliverable. See [How the review is run](#how-the-review-is-run) for why those flags.

### 4. Read the result

A master-plan review checks fidelity to ground truth, technical correctness, soundness of the key decisions, completeness and internal consistency. A decomposition review checks coverage and traceability, round and dependency correctness, the file-conflict matrix, phase self-containment, `file:line` accuracy and right-sizing.

The report lands in the `-o` file. Read it there — the findings are worth reading in the reviewer's own words, and a fix here costs minutes, where the same gap caught in round 2 costs a round.

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

### 3. Run the review

Same shape as Tutorial A: the prompt is printed inline, offered as `<folder>/codex-implementation-review-prompt.md`, and — once saved — offered as a run:

```sh
codex exec --sandbox read-only ${CODEX_MODEL:+-m "$CODEX_MODEL"} \
  -o docs/plans/GH-412/codex-implementation-review.md \
  < docs/plans/GH-412/codex-implementation-review-prompt.md
```

This is the run where **leaving the tree alone matters**: the prompt tells the reviewer to regenerate the diff live rather than trust the embedded snapshot, so anything you change while it works shifts the ground under it. Start the run, then go do something outside the repo.

### 4. Read the findings like findings

**They are claims, not verdicts.** Some are wrong. Verify each against the code before changing anything — accepting a wrong finding costs more than missing a marginal one. The generated prompt makes the reviewer cite `file:line`, separate conformance facts from quality findings from preferences, and say "I couldn't verify X" rather than assume, which is exactly what makes them checkable.

---

## How this relates to `/code-review`

They answer different questions and do not replace each other.

- **`/code-review`** reviews the branch diff on its own merits, inside Claude Code, and can apply its findings with `--fix` or post them to a PR with `--comment`.
- **`plan-implementation-review`** asks whether the changeset built *what the plan specified*, from a separate session with fresh eyes. Plan conformance is not something a diff review can judge, because judging it requires the plan.

`MANUAL.html` runs `/code-review` as step 4 and reaches for these prompts when a genuine second opinion is wanted.

**Don't confuse this with `codex review`.** The Codex CLI has its own review subcommand — `codex review --uncommitted`, or `codex review --base main` — and it is Codex's equivalent of `/code-review`: the diff judged on its own merits. It knows nothing about your plan. Plan conformance needs the plan, the phase files and the spec cross-referenced against the changeset, and that is exactly what the generated prompt carries. The two are complementary; only one of them can tell you a phase was quietly skipped.

## Limits

- **The review is a separate session, always.** The skills can launch it for you, but they never perform it — the fresh-eyes value comes from a reviewer that has no memory of authoring the plan, and that property is the whole separation-of-concerns argument.
- **Running it needs the Codex CLI installed and logged in.** Generating the prompt never does. Without Codex, both skills still produce their deliverable and you take it wherever you like.
- **Artifact detection is convention-based.** It looks for `master-plan.md` / `MASTER_PLAN.md` / `plan.md`, `issue.specs`, `phases/PHASE-*.md`, `tasks.md`, `execute-plan.md` and `handoff.md`. Both skills report what they found before continuing.
- **`plan-review-prompt` reviews plans, not arbitrary code.** For source with no governing plan behind it, use `/code-review`.
