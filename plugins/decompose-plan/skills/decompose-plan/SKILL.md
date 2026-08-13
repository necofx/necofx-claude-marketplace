---
name: decompose-plan
description: This skill should be used to decompose a master implementation plan into atomic, phase-based markdown files organized into sequential and parallel ROUNDS for multi-agent (teammate) execution. Takes a folder path that contains a master plan markdown file and produces individual phase files, a tasks.md progress tracker, an execute-plan.md round-by-round orchestrator prompt, and a handoff.md scaffold for the final code-review hand-off. Tech-stack agnostic — supports .NET, Blazor, React, and Delphi out of the box via the `tech-stack-profiles.md` reference, and is easily extended for other stacks. Trigger when the user invokes `/decompose-plan` with a folder argument, asks to "break down this plan into phases", "decompose this plan", "set up multi-agent execution for this plan", or "phase out this plan for teammates".
---

# Decompose Plan

## Overview

This skill takes a master implementation plan and decomposes it into atomic, phase-based markdown files organized in **rounds** (sequential vs parallel) so several teammates can work in parallel safely under a coordinator. It produces:

1. One markdown file per phase under `<folder>/phases/PHASE-NN-<slug>.md`
2. A `tasks.md` progress tracker that every teammate updates
3. An `execute-plan.md` orchestrator prompt that dispatches teammates round-by-round
4. A `handoff.md` scaffold that the coordinator completes after the last round

The expected input shape is documented in `references/master-plan-format.md`. Keep one completed plan folder around as the in-house reference example.

## Inputs

- **`<folder>`** (required): absolute path to a folder that already contains the master plan markdown file. All outputs are written into this same folder (phases go into `<folder>/phases/`).

## Workflow

Follow these steps in order. Do not skip steps.

### Step 1 — Parse argument and discover the master plan

1. Read `<folder>` from the skill arguments. If absent, ask the user for the folder.
2. List `<folder>` (top-level only).
3. Identify the master plan file by looking, in order, for:
   - `MASTER_PLAN.md`
   - `master-plan.md`
   - `PLAN.md`
   - `plan.md`
   - Any single `*.md` file in the folder if exactly one exists
4. If none or multiple ambiguous candidates exist, ask the user which file is the master plan.
5. If `<folder>/phases/` already exists and is non-empty, ask the user whether to overwrite or abort.

### Step 2 — Load required skills

Invoke these skills via the `Skill` tool BEFORE doing any decomposition work:

1. `Skill(skill="superpowers:using-superpowers")` — establishes skill discipline.
2. `Skill(skill="superpowers:writing-plans")` — informs the structure of each phase file.

If either skill is unavailable, log a warning and continue using `references/` as the canonical fallback.

### Step 3 — Read the canonical master-plan format reference

Read `references/master-plan-format.md` so the analysis in Step 4 is calibrated to what a well-formed master plan contains (rounds, phase index, file-structure map, owner-agent column, pre-merge contract, OOS, open risks).

If the input master plan does not contain these sections, infer them from the prose and explicitly mark inferred fields in a `## Coordination Notes` block of the generated `tasks.md`.

### Step 3.5 — Detect the project's tech stack(s)

Read `references/tech-stack-profiles.md` for the detection precedence and the field shape of each profile. Identify the stack(s) of the project the master plan targets:

1. If the master plan already has a `## Tech Stack` section, use it as authoritative.
2. Otherwise apply the root-marker detection from `tech-stack-profiles.md`.
3. If the project mixes stacks (e.g. .NET API + React SPA, or Delphi desktop + .NET REST server), record the layer-to-stack mapping so per-phase Tech Stack lines can be set correctly in Step 7.
4. If ambiguous, ask the user via `AskUserQuestion`.

Record the resolved stack(s) and which profile(s) to apply. This drives:
- The `Tech Stack:` line of every phase file.
- The Build/Test commands in each phase's Atomic Steps.
- The coding-rules files cited under "Documents to Read".
- The Verification gates listed in each phase and in the final `handoff.md`.

### Step 4 — Locate the project `docs/` folder

The phase files must point teammates at relevant documentation. Find the docs folder by:

1. Walking up from `<folder>` until a sibling directory named `docs` is found.
2. Falling back to `<project-root>/docs` (project root is the nearest ancestor with a `.git` directory).
3. If nothing is found, record `<docs-folder> = (none)` and instruct teammates to derive documentation from source files instead.

List the immediate subdirectories of `<docs-folder>` — these are candidate sources for "Documents to Read" sections in each phase file.

### Step 5 — Analyse the master plan into rounds + phases

A **round** is a set of phases that can run in parallel. Rounds run sequentially. A **phase** is one atomic unit of work executed by one teammate.

Identify rounds + phases by:

1. Reading the master plan end-to-end.
2. Listing every concrete deliverable.
3. Building a dependency graph (which deliverables block which). When the repository has a `.codegraph/` directory, verify the graph against the real code rather than trusting the plan's prose: `codegraph impact <symbol>` shows what a change to a symbol reaches, and `codegraph callers <symbol>` shows who depends on it today. A dependency the plan did not mention is exactly what turns a "parallel" round into a conflict at execution time.
4. Grouping deliverables into atomic phases (single focus, independently verifiable, right-sized — 30 min to 3 h each).
5. Topologically sorting phases into rounds: phases with no unmet dependencies go in Round 0; phases whose dependencies are all in Round 0 go in Round 1; and so on.
6. Round 0 is usually **sequential and small** (one foundation phase); Round 1 is usually the **parallel fan-out**; the last round is usually **integration/verification**.

A good phase is:

- **Single-focus**: one clear outcome (if you want "and", split it).
- **Independently verifiable**: has acceptance criteria that can be checked without running other phases.
- **Right-sized**: 30 min – 3 h. Merge sub-30-min phases; split multi-hour ones.
- **Dependency-explicit**: every cross-phase dependency is named, not implied.

### Step 6 — Build the file-conflict matrix for parallel rounds

For every round with more than one phase, list every file each phase will **Create** or **Modify**, then verify no two phases in the same round touch the same file.

When the repo is indexed, resolve the Modify list against the code instead of guessing from the plan: `codegraph explore "<the area this phase touches>"` returns the real files and their dependents in one call, and `codegraph impact <symbol>` reveals the registration file or call site a phase will have to edit but the plan forgot to list. Those forgotten edits are the usual source of same-round file conflicts.

- If a conflict is unavoidable, document it in `tasks.md` § "Coordination Notes" with a coordination rule: which phase edits first, how the second phase rebases / re-edits.
- Capture the full Create/Modify list per phase — it becomes the "Files" section of each phase file.

### Step 6.5 — Build the skill inventory

The "Skills to Invoke" section of every phase file is filled by matching the phase against the **inventory of skills actually available** in this environment — never by inventing skill names or relying on stale defaults. Build the inventory now so Step 7 has it ready.

**Sources, merged in this order (later sources override earlier on name conflicts):**

1. **Runtime-active skills.** These are the skills currently loaded by the harness (visible in the conversation's available-skills system reminders). They are the most authoritative — they exist AND they are loadable via the `Skill` tool right now. Capture every entry as `{name, description}`.

2. **Project-local skills on disk.** Glob `<project-root>/.claude/skills/**/SKILL.md`. For each match, read just the YAML frontmatter (don't load the full body). Capture `{name, description}` from the frontmatter. Mark any that aren't in the runtime-active set as `source: project-local (may need explicit load)`.

3. **User-global skills (best-effort).** If the platform exposes a user skills directory (e.g. `~/.claude/skills/`), glob it the same way. Skip silently if the directory doesn't exist or isn't accessible. Mark as `source: user-global`.

**Inventory shape (in working memory, no file written):**

```
{
  name: "skill-name",
  description: "first 200 chars from the frontmatter description",
  source: "active" | "project-local" | "user-global",
  hints: { stack: [...], domain: [...], verb: [...] }   // derived in next step
}
```

**Derive hint tags for each entry:**

- **Stack hints** by keyword in the description: `.net|dotnet|c#|csharp|nhibernate|autofac` → `.net`; `blazor|razor|bunit` → `blazor`; `react|typescript|jsx|tsx|next|remix|vite` → `react`; `delphi|pascal|vcl|fmx|firedac` → `delphi`; `python|django|fastapi` → `python`; `sql|postgres|mariadb|sqlite|mssql|oracle|firebird` → `sql`; etc.
- **Domain hints** by keyword: `security|auth|crypto|owasp` → `security`; `perf|performance|benchmark|profil` → `performance`; `debug|diagnose|bug|fix|error` → `debug`; `deploy|docker|kubernetes|k8s|ci|cd|pipeline` → `deploy`; `ui|ux|design|wireframe|component` → `ui`; `database|orm|migration|schema` → `database`; `network|snmp|tcp|http` → `network`; `test|tdd|integration|coverage` → `test`; `architect|design|adr|clean architecture|ddd` → `architecture`; `code-review|review` → `review`.
- **Verb hints** by description preamble: `implement|build|develop|create` → `implement`; `review|audit` → `review`; `debug|fix|diagnose` → `fix`; `optimize|refactor|simplify` → `refactor`; `document|explain` → `document`.

Tags are LOWERCASE and may be multiple per skill (e.g. `dotnet-senior-developer` gets `[stack: .net, blazor; domain: architecture, code-review; verb: implement, refactor]`).

Read `references/skill-matching-heuristics.md` once at this step — it contains the canonical keyword tables and the matching rules used in Step 7.

The inventory is in working memory only; do not write it to disk. It is consumed in Step 7 and discarded when the skill exits.

### Step 7 — Generate phase files

For each phase, create `<folder>/phases/PHASE-NN-<slug>.md` using the template at `references/phase-template.md`. Naming rules:

- `NN` is two-digit zero-padded starting at `00` (Round 0 typically gets `00`).
- `<slug>` is lowercase-kebab from the phase title, max 6 words.

Every phase file MUST include all sections from the template in this order: REQUIRED SUB-SKILL header, Goal, Architecture, Tech Stack, Files, Dependencies, Owner Agent, Risk / Effort, Skills to Invoke, Documents to Read, Pre-execution check, Atomic steps (with TDD discipline), Verification, Notes for downstream phases.

#### Filling "Skills to Invoke" (uses the Step 6.5 inventory)

For each phase, derive these per-phase tags from the master plan + the phase's Files/Goal/Architecture:

- **Stack tag(s)** — from the tech-stack profile(s) chosen in Step 3.5 (e.g. `.net`, `blazor`, `react`, `delphi`). Multiple if the phase straddles stacks.
- **Layer tag(s)** — by inspecting the phase's Files list and Goal: e.g. `data`, `persistence`, `service`, `api`, `ui`, `test`, `infra`, `docs`.
- **Domain tag(s)** — by keyword in the phase's Goal + Architecture: e.g. `security`, `performance`, `database`, `network`, `ui`, `deploy`, `debug`, `architecture`.
- **Verb tag** — `implement` (most code phases), `fix` (bug-fix phases), `refactor` (cleanup phases), `review` (audit phases), `document` (doc phases).

Then build the Skills to Invoke list:

1. **Always-on superpowers (in this order):**
   - `superpowers:using-superpowers` — "establish skill discipline"
   - `superpowers:subagent-driven-development` — "execution discipline for this phase"
   - For any phase delivering code: `superpowers:test-driven-development` — "red-green-refactor for the new tests"
   - `superpowers:verification-before-completion` — "required gate before marking complete"

2. **Matched skills (2–4 of these):**
   - Query the Step 6.5 inventory: for each candidate skill, score it by counting matches between the skill's `hints.{stack,domain,verb}` and the phase's tags.
   - Apply the rules in `references/skill-matching-heuristics.md` § "Scoring" — stack matches weight 3, domain matches 2, verb matches 1; require score ≥ 3 to recommend.
   - Sort by score, take the top 2–4. Prefer `active` source over `project-local` over `user-global` on ties (higher confidence the skill is actually loadable).
   - Reject low-confidence matches rather than padding the list. Empty is fine for trivial phases; the always-on superpowers always carry the floor.

3. **Format every line as `Skill(skill="<name>") — <why, in 6–12 words>`.** The "why" must reference the phase, not parrot the skill description. Examples:
   - `Skill(skill="dotnet-senior-developer-skill") — implementing the new Autofac module and service`
   - `Skill(skill="security-auditor") — reviewing the new auth handler for OWASP issues`
   - `Skill(skill="react-senior-developer-skill") — building the new React component with state management`

4. **Hard rules:**
   - Never invent a skill name not in the inventory.
   - Never include a skill purely "because it might be useful" — if it doesn't pass the score threshold, leave it out.
   - When a phase has no matched skills beyond the always-ons, write `_(no domain-specific matches for this phase; the always-on superpowers cover it)_` instead of inflating the list.
   - When two skills overlap heavily (e.g. `dotnet-senior-developer` and `dotnet-senior-developer-skill`), pick the better-matched one — do not list both.
   - When the inventory is missing a skill the phase obviously needs (e.g. a phase says "build SNMP discovery" but no `network-engineer` skill is registered), record this gap in the per-phase notes section of `tasks.md § Coordination Notes` so the user knows to install the missing skill or accept the gap.

The **Atomic steps** must follow the TDD-discipline pattern when the phase delivers code:

```
- [ ] Step N.0: Claim the phase (set tasks.md row to in_progress)
- [ ] Step N.1: Locate / read the existing surface area being changed
- [ ] Step N.2: Author the first failing test
- [ ] Step N.3: Run; expect FAIL with the exact reason
- [ ] Step N.4: Implement the minimum to make it pass
- [ ] Step N.5: Run; expect PASS
- [ ] Step N.6+: Add more tests (edge cases, ctor guards, cancellation, …)
- [ ] Step N.k-2: Full solution build + test
- [ ] Step N.k-1: TDD proof — temporarily break the impl, re-run, restore
- [ ] Step N.k:   Mark phase complete in tasks.md (Status = completed, Commits = "(pending batch)")
```

**Phases do NOT commit.** Commits are deferred to end-of-round or end-of-plan batches surfaced by the coordinator. Each phase produces a clean, building, fully-tested working tree; the coordinator decides when to surface the batched `git add … && git commit -m "…"` command to the user. The `Commits` column in `tasks.md` records `(pending batch)` until the batch lands; the SHA(s) are filled in by the coordinator after the user runs the batch.

For non-code phases (research, ADR, doc rewrite), omit the TDD steps and substitute domain-appropriate atomic steps.

### Step 8 — Create `tasks.md`

Create `<folder>/tasks.md` from the template at `references/tasks-template.md`. Populate:

- Plan name (from the master plan's H1)
- Today's date (use absolute YYYY-MM-DD; never "today" or weekday names)
- Decomposition strategy in one line
- Phase summary table (every phase, status `pending`, agent `—`, started/finished/commits/pr empty)
- Rounds + dependency graph
- Empty Detailed Progress section per phase
- Empty Coordination Notes section
- Empty Active blockers section
- Empty Decisions section

### Step 9 — Create `execute-plan.md`

Create `<folder>/execute-plan.md` from the template at `references/execute-plan-template.md`. Substitute:

- `{PLAN_NAME}` — master plan's title
- `{PLAN_SLUG}` — lowercase-kebab version of the title (used as the `team_name`)
- `{FOLDER}` — absolute folder path
- The rounds list, with every phase's `PHASE-NN-<slug>.md` filename, title, and owner agent

The orchestrator prompt must instruct the coordinator to:

1. Invoke `superpowers:using-superpowers` and `superpowers:subagent-driven-development`
2. Read `tasks.md`, every `phases/PHASE-*.md`, and the master plan
3. Dispatch each round as a single message with multiple parallel `Agent` calls, named `phase-NN` with `team_name="{PLAN_SLUG}"` and `subagent_type` set to the phase's owner agent (default `general-purpose`)
4. Wait for every phase in a round to reach `completed` before dispatching the next round
5. Between rounds, write a round summary into `tasks.md § Coordination Notes`, surface a batched commit command for the user (optional — they may defer), and advance to the next round autonomously unless the user signals to pause
6. Use `SendMessage(to="phase-NN", ...)` to follow up with running teammates
7. Final round: after the last phase completes, write the `## Final Summary` block in `tasks.md` and populate `handoff.md`

### Step 10 — Create `handoff.md` scaffold

Create `<folder>/handoff.md` from the template at `references/handoff-template.md`. Leave most sections empty with `_(to be filled by the coordinator after Round N)_` placeholders. The coordinator completes this after the last round so the user has a single document to drive code review from.

### Step 11 — Summarise back to the user

After all files are written, report:

- The folder path
- Phase count, round count, parallelism (largest round size)
- Rough wall-clock estimate (sum sequentially across rounds; within a round take the max effort)
- Dependency graph in one or two lines
- Any file-conflict warnings raised in Step 6
- The exact next step: "Open a new conversation, paste the contents of `<folder>/execute-plan.md`, and run it."

Do NOT begin executing the plan in this conversation unless the user explicitly asks — the skill's role ends at decomposition.

## Resources

### `references/master-plan-format.md`
Reference document describing the shape of a well-formed master plan the skill expects as input (rounds, phase index, file-structure map, owner-agent, pre-merge contract, OOS, open risks). Read in Step 3.

### `references/tech-stack-profiles.md`
Profiles for .NET, Blazor, React, and Delphi (build/test commands, test framework, assertion + mocking conventions, code-style notes, conventions location, commit format, root-marker detection). Read in Step 3.5 to pick the correct stack(s); referenced again whenever a template needs a stack-specific value substituted in. Add new profiles here when working on a different stack rather than hardcoding values in the templates.

### `references/skill-matching-heuristics.md`
Canonical keyword tables and scoring rules used in Steps 6.5 + 7 to match `.claude/skills/` entries to each phase. Read in Step 6.5; consulted again in Step 7 when scoring candidates. Update the keyword tables here (not in `SKILL.md`) when adding new domain hints.

### `references/phase-template.md`
Canonical structure for each `PHASE-NN-<slug>.md` file. Read once at the start of Step 7 and apply the same shape to every phase. Stack-specific values (build command, code style, test framework) come from the matching profile in `tech-stack-profiles.md`.

### `references/tasks-template.md`
Canonical structure for the shared `tasks.md` progress tracker. Read at the start of Step 8.

### `references/execute-plan-template.md`
Canonical structure for the round-by-round orchestrator prompt. Read at the start of Step 9. Every placeholder in braces must be substituted with a real value — none should remain in the generated file.

### `references/handoff-template.md`
Canonical structure for the final code-review hand-off document. Read at the start of Step 10. Build/test command lines come from the matching profile in `tech-stack-profiles.md`.

## Notes for the skill operator

- This skill writes files but does NOT execute the plan. Execution happens in a separate conversation triggered by `execute-plan.md`.
- **CodeGraph, when the repo has one.** If a `.codegraph/` directory exists at the repository root, prefer `codegraph_explore` (MCP) or `codegraph explore` / `node` / `callers` / `impact` (shell) over `Glob`/`Grep`/`Read` loops for every "where does this live / who depends on it" question in Steps 4–7. One call returns verbatim line-numbered source plus call paths plus blast radius, which is what decomposition actually needs, at a fraction of the tokens — and it resolves dynamic dispatch that a text search cannot follow. The index lags writes by about a second. If there is no `.codegraph/` directory, use the ordinary tools and do NOT index the repository yourself — that is the user's decision.
- If the master plan is too vague to decompose into clean rounds, stop and ask the user to clarify rather than fabricating phases.
- Always preserve the master plan file as-is; never edit it.
- Prefer fewer, well-scoped phases over many trivial ones. If a phase has fewer than three meaningful tasks, merge it with a neighbour.
- **Tech-stack agnostic.** This skill works on .NET, Blazor, React, and Delphi projects out of the box (and any stack you add to `tech-stack-profiles.md`). All stack-specific build commands, test commands, code-style rules, and conventions live in that one reference — the SKILL.md and templates only refer to it abstractly. To extend to a new stack, add a profile, not a special case here.
- **Mixed-stack projects** (e.g. .NET API + React SPA, or Delphi desktop + .NET REST server) get per-phase Tech Stack lines. The skill tags each phase with the stack of the files it touches.
- **Coding-rules files.** When the project has a `.claude/rules/*.md` directory (or equivalent — `CONTRIBUTING.md`, `STYLE.md`, `.editorconfig`), cite the specific rule files in each phase's "Documents to Read" section instead of restating the rules. If the project has none, fall back to the profile's general guidance and note that fact in the per-phase notes of `tasks.md`.
- **Git constraint and commit cadence.** Phases never request commits themselves — the templates generated by this skill omit the per-phase commit step. Commits are deferred to **batches** (typically end-of-round or end-of-plan) surfaced by the coordinator to the user. This keeps multi-agent execution flowing without per-phase interruptions. The user remains the only actor who runs `git commit`; the coordinator simply assembles copy-pasteable commands once a coherent batch is green. When the project enforces read-only git for Claude Code (e.g. via `.claude/rules/git-workflow.md`), the batch commands are emitted as text — the user runs them. When the project lets agents commit, the coordinator may run the batch directly.
- **Recommended teammate-side superpowers.** Always include `superpowers:using-superpowers`, `superpowers:subagent-driven-development`, `superpowers:verification-before-completion`. Add `superpowers:test-driven-development` for code phases. Add stack-specific skills (e.g. `dotnet-senior-developer-skill`, `delphi-senior-developer-skill`, frontend skills for React, Blazor-specific skills if available) per the matching profile.
