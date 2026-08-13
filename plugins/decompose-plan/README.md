# decompose-plan

Turns a master plan into something several agents can execute at once. It reads the plan, builds a dependency graph, groups deliverables into atomic phases, sorts those phases into **rounds** — a round is a set of phases that run in parallel, rounds run sequentially — and verifies with a file-conflict matrix that no two phases in the same round touch the same file.

Then it writes one file per phase, a shared status board, the coordinator prompt you paste into a fresh conversation, and the hand-off scaffold your reviewer will read.

## Credits

**The workflow is not ours.** It was designed by someone else, who uses it daily and shared their files directly, and it is published here with their permission. The structure, the reasoning and nearly all of the prose are theirs.

The only change in this plugin is that the original's `{{TICKET_PREFIX}}` placeholder is gone — plan ids now come from whatever [`create-master-plan`](../create-master-plan/) derived (`GH-412` by default). Nothing else was touched.

`MANUAL.html` in this folder is their manual for the complete five-step workflow, and it explains the reasoning this README only summarises.

---

## Setup

### 1. Install the plugin

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install decompose-plan@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH — pass `https://github.com/necofx/necofx-claude-marketplace.git` instead.

### 2. Install `superpowers` — required

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install superpowers@claude-plugins-official
```

This skill loads `superpowers:writing-plans` to shape each phase file. More importantly, the phase files and the coordinator prompt it *generates* instruct the executing agents to invoke `subagent-driven-development`, `test-driven-development`, `verification-before-completion` and `dispatching-parallel-agents`. Without the plugin the skill logs a warning and falls back to its own references — but the real loss lands at execution time, because `verification-before-completion` is the gate that stops an agent claiming a phase is done when it is not.

### 3. Install the specialist agents — recommended

Every phase names an `owner_agent`, used by the coordinator as `subagent_type`:

```
/plugin marketplace add wshobson/agents
```

That marketplace (`claude-code-workflows`, MIT) ships the generic ones the skill cites — `code-reviewer`, `security-auditor`, `sql-pro`, `performance-engineer`, `backend-architect`, `architect-review`, `debugger` — inside topical bundles. Install the bundles matching your work. Without them, phases fall back to `general-purpose`, which the phase template explicitly allows.

### 4. Optional: CodeGraph

```sh
npm install -g @colbymchenry/codegraph
codegraph install -y -t claude -l global
codegraph telemetry off
cd /path/to/your/repo && codegraph init
```

Used in two places that matter here: checking the plan's claimed dependencies against real callers, and resolving each phase's true modify-list. **The registration file the plan forgot to mention is the usual source of a same-round conflict**, and `codegraph impact` finds it where prose does not. Every instruction is conditional on a `.codegraph/` directory existing, so an unindexed repo just costs more tokens.

### 5. Restart

Skills load at session start.

---

## Tutorial: from a plan to five agents

### 1. Decompose

```
/decompose-plan docs/plans/GH-412
```

Point it at a folder that already contains a master plan — it looks for `MASTER_PLAN.md`, `master-plan.md`, `PLAN.md`, `plan.md`, or the single `*.md` if exactly one exists, and asks if that is ambiguous. Everything it writes lands in that same folder. If `phases/` already exists and is non-empty, it asks before overwriting.

It reads the plan end to end, lists every deliverable, builds the dependency graph, and groups the work into phases. A **good phase** is single-focus (an "and" in the goal means split it), independently verifiable without running the other phases, 30 minutes to 3 hours, and names every cross-phase dependency rather than implying it. Then it topologically sorts them: Round 0 is usually one foundation phase, Round 1 the wide parallel fan-out, the last round integration and verification.

**Two mechanisms do the real work.** The *file-conflict matrix* lists each phase's create/modify files for every multi-phase round and verifies no two overlap — an unavoidable overlap gets an explicit coordination rule in `tasks.md` naming who edits first. *Skill matching* fills each phase's "Skills to Invoke" from an inventory of what actually exists (runtime-active → `.claude/skills/**/SKILL.md` → user-global), scoring candidates `3×stack + 2×domain + 1×verb`, `+2` when runtime-active, `−5` on conflicting stacks; recommended at ≥3, capped at 4. No match means the always-on skills plus an explicit note, never a padded list.

It finishes with a summary: phase count, round count, the largest round, a rough wall-clock estimate, the dependency graph, and any conflict warnings.

### 2. Read the output before running anything

| File | What to check |
|---|---|
| `phases/PHASE-NN-<slug>.md` | Are these phases you would have written yourself? Does the Files list look complete — or is a registration file missing? |
| `tasks.md` | The `## Coordination Notes` section flags anything the skill had to *infer* because the master plan did not say. Read those flags. |
| `execute-plan.md` | The round list, then the Coordinator Prompt you paste. |
| `handoff.md` | Empty scaffold; the coordinator fills it after the last round. |

**Vague or overlapping phases mean an underspecified master plan.** Do not patch the phases — go back and sharpen the plan. Patching phases fixes the symptom in one place and leaves it everywhere else.

### 3. Execute — in a fresh conversation

Open a new conversation and paste the entire **Coordinator Prompt** block from `execute-plan.md`. The coordinator needs a near-empty context window: it holds the whole plan, every phase file, and every agent's report.

The coordinator then loops:

1. Reads the master plan, `tasks.md` and every phase file — the whole shape first.
2. Dispatches every ready phase in the round **in a single message**, one `Agent` call each, named `phase-NN` with the phase's owner agent as `subagent_type`. This is what makes them genuinely concurrent.
3. Waits for the whole round to reach `completed`. You can check in on a running agent via `SendMessage(to="phase-NN", …)`.
4. Runs the project's full build and test gate to confirm the phases integrate, and writes a round summary into `tasks.md`.
5. Surfaces a **batched commit command** as copy-pasteable text and advances without waiting for you to run it.

### The TDD shape inside a phase

Locate the surface area → one failing test → confirm it fails for the exact expected reason → minimum implementation → confirm it passes → edge cases one at a time → full build and test gate → **TDD proof: deliberately break the implementation, confirm the tests that should fail do fail, restore.**

That last step is what separates tests that verify behaviour from tests that merely execute code. In a run where no human watches any individual phase, it is most of what "green" is worth.

**Phases never commit.** Each leaves a clean, tested tree and writes `(pending batch)`; the coordinator batches commits at end-of-round and hands them to you. Parallel agents committing independently produce interleaved partial commits nobody can review, and per-phase approval stalls the run on every finish.

### 4. Close out

After the last round: real commit SHAs and the final summary in `tasks.md`, every section of `handoff.md` filled — **especially the deviations from the plan**, which is where reviewers spend their attention and where unexamined assumptions hide — and anything durable folded into your `.claude/rules/` rather than left in a plan folder nobody reopens.

Then [`plan-review`](../plan-review/) for a second opinion on what actually landed.

---

## When it goes wrong

| Symptom | Cause and fix |
|---|---|
| Vague or overlapping phases | An underspecified master plan. Sharpen the plan, not the phases. Read the inferred-section flags in `tasks.md § Coordination Notes`. |
| Two phases in one round hit the same file | The matrix missed it. Move one phase to the next round, or add an explicit coordination rule naming who edits first. |
| The coordinator dispatches sequentially | All phases in a round must go in **one message** with one `Agent` call each. Sequential dispatch forfeits the entire point of rounds. |
| An agent claims done, the build is broken | The verification gate was skipped. Check the phase's Verification section names your real commands, and that `superpowers:verification-before-completion` resolves. |
| A phase goes silent | Ask it to summarise and update `tasks.md`. If genuinely blocked it should set `Status = blocked` and fill Active blockers; the coordinator then re-dispatches, resolves, or drops it. |
| Agents ignore your conventions | No rule files, or the phase files cite none. Check a phase's "Documents to Read" — an empty list means there was nothing to cite. Writing `.claude/rules/*.md` is the highest-leverage thing you can do here, and nobody can ship it for you. |

## Limits

- **Four owner agents are the workflow author's own, not public.** `dotnet-senior-developer`, `blazor-frontend-developer`, `delphi-senior-developer` and `react-senior-developer` do not come from `wshobson/agents` and are not in this package. Phases naming them fall back to `general-purpose`. Write your own with those names if you work in those stacks — the matcher scores against the agent description, so name the stack and domain explicitly in it.
- **It does not execute the plan.** Execution happens in a separate conversation driven by `execute-plan.md`.
- **It never edits the master plan.** The input is preserved as-is.
