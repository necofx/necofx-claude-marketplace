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

Every phase file carries an **Owner Agent** line. The coordinator passes that name straight to `Agent(subagent_type=…)`. If the name does not resolve to an installed agent, the call falls back to `general-purpose` — the run still works, it just loses the specialist's system prompt. So this step is: install the bundles that ship the agent names your phases will actually cite.

```
/plugin marketplace add wshobson/agents
```

That is Seth Hobson's `claude-code-workflows` marketplace (MIT, 95 bundles). Agents ship **inside topical bundles**, never individually — you install `comprehensive-review` and `code-reviewer` comes with it. The install syntax is therefore:

```
/plugin install <bundle>@claude-code-workflows
```

#### Worked example — a Java + Docker + Kubernetes service with some Python tooling

Paste these nine lines:

```
/plugin install jvm-languages@claude-code-workflows
/plugin install python-development@claude-code-workflows
/plugin install kubernetes-operations@claude-code-workflows
/plugin install cloud-infrastructure@claude-code-workflows
/plugin install cicd-automation@claude-code-workflows
/plugin install comprehensive-review@claude-code-workflows
/plugin install observability-monitoring@claude-code-workflows
/plugin install database-design@claude-code-workflows
/plugin install error-debugging@claude-code-workflows
```

Line by line — what each one buys and which phases will name it:

| Bundle | Agents it installs | Phases that cite them |
|---|---|---|
| `jvm-languages` | `java-pro`, `scala-pro`, `csharp-pro` | Everything under `src/main/java` |
| `python-development` | `python-pro`, `django-pro`, `fastapi-pro` | The Python tooling / scripts phases |
| `kubernetes-operations` | `kubernetes-architect` | Phases touching `helm/`, `k8s/`, manifests |
| `cloud-infrastructure` | `cloud-architect`, `terraform-specialist`, `kubernetes-architect`, `network-engineer`, `service-mesh-expert`, `hybrid-cloud-architect`, `deployment-engineer` | IaC and cluster-topology phases |
| `cicd-automation` | `deployment-engineer`, `devops-troubleshooter`, `kubernetes-architect`, `terraform-specialist`, `cloud-architect` | `Dockerfile` and pipeline phases |
| `comprehensive-review` | `code-reviewer`, `architect-review`, `security-auditor` | The final review phase of almost every plan |
| `observability-monitoring` | `performance-engineer`, `observability-engineer`, `database-optimizer`, `network-engineer` | Metrics / tracing / SLO phases |
| `database-design` | `sql-pro`, `database-architect` | Schema and query phases |
| `error-debugging` | `debugger`, `error-detective` | None — you dispatch these by hand when a round breaks |

Four things that save you a reinstall:

- **There is no Docker agent.** Containers are covered by `deployment-engineer` (in `cicd-automation` and `cloud-infrastructure`) and by the container-scanning half of `security-scanning`. A phase whose files are a `Dockerfile` should name `deployment-engineer`.
- **Agents are duplicated across bundles**, so you rarely need the bundle you first think of. `security-auditor` ships in five (`comprehensive-review`, `security-scanning`, `backend-development`, `full-stack-orchestration`, `security-compliance`); `code-reviewer` in seven. Install the smallest bundle that carries the name and stop.
- **`developer-essentials` is a trap here.** Despite the name, its only agent is `monorepo-architect` — everything else in it is skills. Install it for the skills, never to satisfy an Owner Agent line.
- **Minimum viable set:** `comprehensive-review` plus the one language bundle for your stack. That covers the review phase and the implementation phases; everything else is refinement.

Without any of them every phase falls back to `general-purpose`, which the phase template explicitly allows.

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

### 0. Where this sits, and what carries the handoff

This is step 2 of five. Nothing is passed between the steps by hand — **the plan folder is the handoff.** Every skill reads and writes the same directory, so "giving the next step the plan" is just naming that folder again.

```
docs/plans/active/GH-412/          ← name this folder at every step
├── issue.specs             written by create-master-plan
├── master-plan.md          written by create-master-plan   ← input to this plugin
├── phases/PHASE-NN-*.md    written by decompose-plan
├── tasks.md                written by decompose-plan, then updated live by every agent
├── execute-plan.md         written by decompose-plan, pasted by you
└── handoff.md              scaffolded by decompose-plan, filled by the coordinator at the end
```

| # | You run | Where | Produces |
|---|---|---|---|
| 1 | `/create-master-plan 412` | any conversation | `issue.specs`, `master-plan.md` |
| 2 | `/decompose-plan docs/plans/active/GH-412` | same conversation is fine | `phases/`, `tasks.md`, `execute-plan.md`, `handoff.md` |
| 2.5 | `/plan-review-prompt` → paste output into Codex | any conversation | findings you fold back into the plan |
| 3 | paste the Coordinator Prompt (below) | **a fresh conversation — mandatory** | the code, plus `tasks.md` and `handoff.md` filled in |
| 4 | `/plan-implementation-review` → paste output into Codex | any conversation | findings on what actually landed |

Only one of those transitions is load-bearing: **step 3 must start in an empty conversation.** The rest can share one.

Step 2.5 is the cheap review and the one people skip. It reviews the plan *before* anything is built — a wrong phase boundary caught there costs you minutes, the same boundary caught in round 2 costs a whole round plus a confused human. Step 4 is its post-implementation sibling: same folder, but it judges the real `git diff` against what the plan promised. Both live in [`plan-review`](../plan-review/) and neither performs the review itself — they generate a self-contained prompt you hand to a *different* agent, on the theory that the author of a plan is a poor reviewer of it.

### 1. Decompose

```
/decompose-plan docs/plans/active/GH-412
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

**What "the Coordinator Prompt block" means.** `execute-plan.md` has three parts, and only the middle one is for the machine:

1. A round list at the top — `Round 0`, `Round 1`, … with each phase and its owner agent. **For you**, so you can see the shape before committing to it.
2. A `## Coordinator Prompt` heading followed by **one fenced code block, roughly 170 lines.** *This is the thing you paste.* It opens with `You are the coordinator for the implementation of "…"` and closes with the "recommended next step" line.
3. `## Tips for the coordinator` at the bottom — **also for you.** Out-of-band commentary about why the rounds work the way they do. Do not paste it; appending it dilutes the instruction the coordinator is following.

So: open the file, select everything *between* the two ``` fences under `## Coordinator Prompt`, and paste that into a new conversation. Or lift it from the command line:

````bash
awk '/^## Coordinator Prompt/{f=1;next} f&&/^```/{c++;next} f&&c==1' \
  docs/plans/active/GH-412/execute-plan.md
````

Append `| clip.exe` on WSL, `| pbcopy` on macOS, or `| xclip -sel c` on Linux to skip the scrolling. Paste it as your very first message — nothing before it, no "here's the plan" preamble.

**Why the conversation must be fresh:** the coordinator ends up holding the master plan, every phase file, and every teammate's full report simultaneously. Starting it in a window that already contains your planning discussion means it hits compaction mid-run — and a coordinator that has forgotten round 1's deviations will happily dispatch round 2 on top of them.

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

- **Four owner agents are the workflow author's own, not public.** `dotnet-senior-developer`, `blazor-frontend-developer`, `delphi-senior-developer` and `react-senior-developer` do not come from `wshobson/agents` and are not in this package. Phases naming them fall back to `general-purpose`. Write your own with those names if you work in those stacks — the matcher scores against the agent description, so name the stack and domain explicitly in it. The Java, Python and Kubernetes owner agents are the exception: `java-pro`, `python-pro` and `kubernetes-architect` are real public agents, so those phases resolve out of the box once the bundles above are installed.
- **It does not execute the plan.** Execution happens in a separate conversation driven by `execute-plan.md`.
- **It never edits the master plan.** The input is preserved as-is.
