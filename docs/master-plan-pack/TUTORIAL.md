# The master-plan pack — one ticket, end to end

Three plugins — [`create-master-plan`](../../plugins/create-master-plan/), [`decompose-plan`](../../plugins/decompose-plan/), [`plan-review`](../../plugins/plan-review/) — are one workflow. Each plugin's README documents that plugin. **This document is the worked example that joins them**: a single ticket, on a single repository, from an empty machine to a merged change.

Everything here is one continuous story. The repository, the ticket and the artifacts are invented; every command, file name, template section and agent name is real, and comes from the skills as they ship.

---

## The example project

A payments platform in one repository, deliberately polyglot — because the interesting failures in a multi-agent run happen *between* layers, not inside one.

```
acme-payments/
├── payments-api/                  Java 21 · Spring Boot 3 · Gradle module :payments-api
│   ├── src/main/java/com/acme/payments/…
│   ├── src/main/resources/application.yml
│   └── src/test/java/com/acme/payments/…      JUnit 5 · AssertJ · Mockito · Testcontainers
├── reconciler/                    Python 3.12 worker · pyproject.toml · pytest
│   └── src/reconciler/…
├── db/migrations/                 Flyway SQL · PostgreSQL 16
├── deploy/
│   ├── docker/                    Dockerfile.api · Dockerfile.reconciler
│   └── helm/payments/             Helm chart · values.yaml · templates/
├── docs/                          architecture notes, runbooks, ADRs
└── .claude/rules/                 the team's coding rules
```

**The ticket.** GitHub issue **#412 — "Support partial refunds"**. Today only full refunds exist. Finance needs to refund part of a payment, more than once, up to the original amount. It crosses every layer: a schema change, Java domain and API work, a Python reconciler that must stop treating a refund as terminal, Helm configuration for a new feature flag, and an end-to-end proof.

That is the shape this workflow is for. A one-file change does not need it.

---

## The five steps, and what carries state between them

Nothing is handed between steps by hand. **The plan folder is the handoff** — every step reads and writes `docs/plans/GH-412/`, so "give the next step the plan" is just naming that folder again.

```mermaid
flowchart TD
    T["GitHub issue #412"] --> S1

    subgraph S1["1 · create-master-plan"]
        A1["issue.specs"]
        A2["master-plan.md"]
    end

    S1 --> S2

    subgraph S2["2 · decompose-plan"]
        B1["phases/PHASE-00…05.md"]
        B2["tasks.md"]
        B3["execute-plan.md"]
        B4["handoff.md — empty scaffold"]
    end

    S2 -. optional .-> R1["2.5 · plan-review-prompt<br/>review the plan before building it"]
    R1 -. findings .-> S1

    S2 --> S3["3 · paste the Coordinator Prompt<br/>FRESH conversation — mandatory"]
    S3 --> C1["the code · tasks.md filled · handoff.md filled"]

    C1 --> S4["4 · /code-review<br/>the diff on its own merits"]
    C1 -. optional .-> R2["5 · plan-implementation-review<br/>the diff against the plan"]
    S4 --> M["commit · merge"]
    R2 -. findings .-> M
```

| # | You run | Conversation | Produces |
|---|---|---|---|
| 1 | `/create-master-plan 412` | any | `issue.specs`, `master-plan.md` |
| 2 | `/decompose-plan docs/plans/GH-412` | the same one is fine | `phases/`, `tasks.md`, `execute-plan.md`, `handoff.md` |
| 2.5 | `/plan-review-prompt` | any | findings you fold back into the plan — **optional** |
| 3 | paste the Coordinator Prompt | **a fresh one — mandatory** | the code |
| 4 | `/code-review` | any | findings on the diff |
| 5 | `/plan-implementation-review` | any | findings on the diff *against the plan* — **optional** |

Only one transition is load-bearing: **step 3 must start in an empty conversation.** The coordinator ends up holding the master plan, every phase file and every teammate's report at once. Start it in a window that already contains your planning discussion and it hits compaction mid-run — and a coordinator that has forgotten round 1's deviations will cheerfully dispatch round 2 on top of them.

---

# Part 0 · Setup

Once per machine, except the last step.

### 0.1 · The marketplace and the three plugins

```
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install create-master-plan@necofx
/plugin install decompose-plan@necofx
/plugin install plan-review@necofx
```

If the first line fails to clone, the `owner/repo` shorthand is trying SSH. Pass the HTTPS URL instead — `https://github.com/necofx/necofx-claude-marketplace.git` — or export `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1`.

### 0.2 · `superpowers` — required, not optional

```
/plugin marketplace add anthropics/claude-plugins-official
/plugin install superpowers@claude-plugins-official
```

Steps 1 and 2 degrade gracefully without it. **Step 3 does not.** The phase files instruct each teammate to invoke `superpowers:verification-before-completion`, and that is the gate standing between "the agent says the phase is done" and "the phase is done". In a run where nobody watches any individual phase, it is most of what green is worth.

### 0.3 · The specialist agents

Every phase file carries an **Owner Agent** line, which the coordinator passes straight to `Agent(subagent_type=…)`. An unresolvable name silently falls back to `general-purpose` — the run survives, it just loses the specialist's system prompt. So install the bundles that ship the names your phases will cite.

```
/plugin marketplace add wshobson/agents
```

For this project's stack:

```
/plugin install jvm-languages@claude-code-workflows
/plugin install python-development@claude-code-workflows
/plugin install kubernetes-operations@claude-code-workflows
/plugin install cicd-automation@claude-code-workflows
/plugin install database-design@claude-code-workflows
/plugin install comprehensive-review@claude-code-workflows
```

| Bundle | Buys you | Phases that will name it |
|---|---|---|
| `jvm-languages` | `java-pro` | everything under `payments-api/src/main/java` |
| `python-development` | `python-pro` | the reconciler |
| `kubernetes-operations` | `kubernetes-architect` | `deploy/helm/**` |
| `cicd-automation` | `deployment-engineer` | `deploy/docker/**` — **there is no Docker agent**; this is it |
| `database-design` | `sql-pro`, `database-architect` | the Flyway migration |
| `comprehensive-review` | `code-reviewer`, `security-auditor`, `architect-review` | the closing review of almost every plan |

Two traps worth knowing before you install too much: agents are duplicated across bundles, so install the smallest bundle carrying the name and stop; and `developer-essentials` ships exactly one agent despite the name.

### 0.4 · CodeGraph — optional, and it earns its keep here

```sh
npm install -g @colbymchenry/codegraph    # a real global install; npx-only silently fails
codegraph install -y -t claude -l global  # the agent id is `claude`, not `claude-code`
codegraph telemetry off                   # telemetry ships ENABLED
cd ~/src/acme-payments && codegraph init
```

Used at four separate points in this workflow, and each one is a place the workflow otherwise guesses:

```mermaid
flowchart LR
    CG[".codegraph/ index"]
    CG --> U1["step 1 · the interview grounds<br/>its questions in real code"]
    CG --> U2["step 2 · resolves each phase's<br/>true modify-list"]
    CG --> U3["step 3 · each teammate locates<br/>its surface area in one call"]
    CG --> U4["steps 2.5 and 5 · the skill builds the<br/>reading list from the graph…"]
    U4 --> U5["…and the reviewer verifies<br/>blast radius the same way"]
```

The one that pays for the install is **step 2**: `codegraph impact` finds the registration file the plan forgot to mention, and a forgotten registration file is the usual cause of two phases in the same round colliding.

Every CodeGraph instruction in every skill is conditional on `.codegraph/` existing, so an unindexed repo simply costs more tokens. **No skill will ever index for you** — it writes hundreds of megabytes, so it stays your decision.

### 0.5 · Restart

Skills load at session start. Open a new conversation before the first `/create-master-plan`.

---

# Part 1 · From ticket to plan

```
/create-master-plan 412
```

Takes `412`, `#412`, or the issue URL. GitHub is the default and uses the `gh` CLI, so `gh auth status` should be happy. Jira and Linear are adapter profiles; pasted free-form text also works, in which case step 1 is the only step that ever knew there was a tracker.

### What it does before it asks you anything

1. Creates `docs/plans/GH-412/`. If that folder already exists with files, it stops and asks — overwrite, merge to `.bak.<timestamp>`, or abort. It will not clobber silently.
2. Fetches the issue with every field and comment, downloads attachments into `attachments/`, follows up to 5 linked items and quotes up to 3 cited documents in full. **A referenced merged PR is the most valuable thing it can find** — its file list is how "this ticket is gap-closure, not a build" gets discovered without grepping blind.
3. Globs `docs/**/*.md` with no exclusions and no cap, grepping for the ticket key, the component names and the distinctive nouns. Long output here is correct: this is the step that surfaces the internal document nobody remembered.
4. Writes `issue.specs`. **Before** the interview, deliberately — everything in that file now counts as known, so an interview question the file already answers is a defect you can point at.

```mermaid
sequenceDiagram
    autonumber
    participant U as You
    participant S as create-master-plan
    participant GH as gh CLI · GitHub
    participant D as the repo · docs/ · .codegraph/
    participant F as docs/plans/GH-412/

    U->>S: /create-master-plan 412
    S->>F: create the folder — ask first if it exists
    S->>GH: issue #412, all fields + comments
    S->>GH: up to 5 linked items · #388 merged PR
    S->>GH: up to 3 cited documents, quoted in full
    GH-->>S: the raw record
    S->>F: attachments/ downloaded
    S->>D: glob docs/**/*.md — no cap, no exclusions
    D-->>S: payments-state-machine.md · reconciler.md
    S->>F: write issue.specs — BEFORE asking anything

    Note over S,D: everything above now counts as known —<br/>a question it already answers is a defect

    loop several rounds — do not rush this
        S->>D: codegraph explore — ground the question in real code
        S->>U: AskUserQuestion, up to 4 at a time
        U-->>S: answers
    end

    S->>U: here is the outline — write it / edit it / ask more
    U-->>S: edit the outline first
    S->>F: write master-plan.md
    S->>U: folder, files, sizes, next command
```

### `issue.specs` — the dossier

```markdown
# GH-412 — Support partial refunds

**Status:** open
**Type:** feature
**Priority:** _(not supplied by this source)_
**Author:** finance-ops
**Assignee:** sebas
**Components:** _(not supplied by this source)_
**Labels:** payments, api, needs-migration
**Source:** github
**Fetched at:** 2026-08-14 09:12
**Plan folder:** `docs/plans/GH-412/`

## 1. Description

Today `POST /payments/{id}/refund` refunds the full captured amount and marks the
payment REFUNDED. Finance needs partial refunds: several refunds against one
payment, each with its own amount, until the sum reaches the captured total.

## 2. Acceptance Criteria

- A payment can be refunded partially, more than once
- The sum of refunds can never exceed the captured amount
- The reconciler no longer treats the first refund as terminal
- Existing full-refund clients keep working unchanged

## 4. Linked issues

### #388 · relates-to · merged
**Title:** Add idempotency keys to payment mutations
Introduced `IdempotencyKeyFilter` and the `idempotency_keys` table. Touched
`PaymentController.java`, `IdempotencyKeyFilter.java`, `V9__idempotency.sql`.
Any new mutating endpoint inherits this filter — GH-412's refund endpoint must
carry an idempotency key.

## 7. Related local docs

- `docs/architecture/payments-state-machine.md` — matched: `REFUNDED`, `payment` —
  excerpt: "CAPTURED → REFUNDED is terminal; the reconciler asserts it"
- `docs/runbooks/reconciler.md` — matched: `reconciler` —
  excerpt: "the worker re-reads events from the last checkpoint on restart"

## 8. Context Gaps

- GitHub supplies no priority or components field; the interview should confirm
  urgency and which teams own the reconciler.
- The ticket never says whether a partial refund can be reversed. Ask.
```

That last section is the one people skim past and shouldn't. **An absence is data.** Recording *why* something is empty stops a later reader from mistaking thinness for laziness.

### Then the interview — and this is the part that decides quality

Questions arrive through `AskUserQuestion` with real options, batched up to four at a time, over a coverage matrix: goal and value, constraints, scope boundary, technical implementation, edge cases, tradeoffs, acceptance criteria, testing strategy, validation gates. Anything `issue.specs` already answered is skipped, and each question is grounded in the code first — on an indexed repo, with a single `codegraph explore` call rather than a read-every-match loop.

For GH-412 it will ask things like: can a partial refund be reversed? What happens to the payment state when refunds sum to exactly the captured amount? Is the reconciler allowed to lag, or must it be strictly consistent? Which of these gates does CI actually run?

**Answer properly.** Everything downstream derives from this. A vague requirement here becomes several agents confidently building the wrong thing *in parallel*, two steps later. Expect several rounds — two rounds only covers the obvious.

Before writing, it shows you the outline and offers: *looks good — write it* / *edit the outline first* / *ask more questions*. Use the middle one freely; it is far cheaper than fixing the plan afterwards.

### `master-plan.md` — the excerpt that matters

```markdown
# GH-412 — Master Plan: Support partial refunds

## Tech Stack

- **Java / JVM (Gradle)** — covers the API and domain
  - Java 21, Spring Boot 3.3
  - Build: `./gradlew :payments-api:build`
  - Test: JUnit 5 + AssertJ + Mockito; Testcontainers for the Postgres integration tests
  - Coding rules: `.editorconfig`, `config/checkstyle/checkstyle.xml`, `.claude/rules/*.md`
- **Python** — covers the reconciler worker
  - Python 3.12, pytest
  - Build/test: `uv run pytest reconciler/tests`
  - Coding rules: `pyproject.toml` [tool.ruff], `.claude/rules/python.md`
- **PostgreSQL 16** — Flyway migrations under `db/migrations/`
- **Docker / Kubernetes** — Helm chart at `deploy/helm/payments`

## Out of scope

- Reversing a partial refund — confirmed out of scope in the interview; it needs
  a policy decision finance has not made.
- Multi-currency refunds — the platform is single-currency until GH-455.

## Technical Requirements

### Data layer
- A `refunds` table exists with `payment_id`, `amount_minor`, `created_at`,
  `idempotency_key`, and a constraint that the sum per payment cannot exceed
  the payment's captured amount.

### Service layer
- `RefundService` supports repeated partial refunds and rejects any refund that
  would breach the captured total, returning a domain error rather than throwing.

### API layer
- `POST /payments/{id}/refunds` accepts an amount, carries an idempotency key
  per GH-388, and returns 409 on over-refund.

### Worker
- The reconciler treats `refund.partial` as non-terminal and only closes a
  payment when the refunded sum equals the captured amount.

### Cross-cutting
- The feature is behind `payments.partial-refunds.enabled`, defaulted off, and
  settable per environment from the Helm values.

## Validation & Testing

- [ ] `./gradlew :payments-api:build` → 0 warnings, 0 errors
- [ ] `./gradlew spotlessCheck check` → clean
- [ ] `uv run pytest reconciler/tests` → green
- [ ] `helm template deploy/helm/payments` → renders, and the new key is present
- [ ] Flyway migration applies to a clean database and to a copy of staging
```

### Before you move on, check exactly two things

**Is the implementation outline at chunk level?** It should say "add the refunds endpoint", not name functions. Pre-decomposing here produces phases that collide later.

**Do the validation gates name your real commands?** If the plan says `pytest` and your project needs `uv run pytest`, fix it now. An assumed verification command does not fail at planning time — it fails in round 2, when four phases already rest on it and correcting it means redoing the decomposition.

---

# Part 2 · From plan to phases and rounds

```
/decompose-plan docs/plans/GH-412
```

It reads the plan end to end, lists every deliverable, builds the dependency graph, and groups the work into phases. A **good phase** is single-focus — an "and" in the goal means split it — independently verifiable, 30 minutes to 3 hours, and names every cross-phase dependency instead of implying it.

```mermaid
sequenceDiagram
    autonumber
    participant U as You
    participant S as decompose-plan
    participant M as master-plan.md
    participant CG as codegraph
    participant I as skill inventory
    participant F as docs/plans/GH-412/

    U->>S: /decompose-plan docs/plans/GH-412
    S->>M: read end to end
    M-->>S: deliverables, layers, validation gates
    S->>S: dependency graph → atomic phases → topological sort into rounds

    S->>CG: impact RefundService — what does this really reach?
    CG-->>S: callers + the registration file the plan never named
    Note over S,CG: this is the call that prevents<br/>a same-round file collision

    S->>S: file-conflict matrix per multi-phase round
    Note over S: Round 1 — PHASE-01 and PHASE-03 both want<br/>application.yml → write a coordination rule

    S->>I: what skills exist here? runtime → .claude/skills → user-global
    I-->>S: scored 3×stack + 2×domain + 1×verb, ≥3 recommended, max 4

    S->>F: phases/PHASE-00…05.md
    S->>F: tasks.md — status board + matrix + inferred-section flags
    S->>F: execute-plan.md — the coordinator prompt
    S->>F: handoff.md — empty scaffold
    S->>U: 6 phases · 4 rounds · largest round 3 · ~7 h sequential, ~4 h parallel
```

Then it topologically sorts them into rounds. **Phases in a round run in parallel; rounds run sequentially.**

```mermaid
flowchart TD
    R0["ROUND 0 — sequential"]
    P00["PHASE-00 · Flyway migration + refunds table<br/>owner: sql-pro"]
    R0 --> P00

    P00 --> R1["ROUND 1 — 3 in parallel, no shared files"]
    R1 --> P01["PHASE-01 · RefundService partial-refund domain<br/>owner: java-pro"]
    R1 --> P02["PHASE-02 · reconciler stops treating refunds as terminal<br/>owner: python-pro"]
    R1 --> P03["PHASE-03 · Helm values + ConfigMap for the flag<br/>owner: kubernetes-architect"]

    P01 --> R2["ROUND 2 — sequential"]
    P02 --> R2
    P03 --> R2
    R2 --> P04["PHASE-04 · POST /payments/id/refunds + OpenAPI<br/>owner: java-pro"]

    P04 --> R3["ROUND 3 — sequential"]
    R3 --> P05["PHASE-05 · Testcontainers E2E + image build + helm template<br/>owner: deployment-engineer"]
```

Two mechanisms do the real work here, and both are worth understanding because both are things you will be asked to check.

**The file-conflict matrix.** For every multi-phase round it lists each phase's create/modify files and verifies no two overlap. In this plan it catches a real one: Phase 01 needs the feature-flag property in `payments-api/src/main/resources/application.yml`, and Phase 03 wants to template the same key from Helm. Same round, same file. An unavoidable overlap does not stop the run — it gets an explicit coordination rule written into `tasks.md` naming who edits first.

**Skill matching.** Each phase's "Skills to Invoke" is filled from an inventory of what actually exists on your machine — runtime-active, then `.claude/skills/**/SKILL.md`, then user-global — scoring candidates `3×stack + 2×domain + 1×verb`, `+2` when runtime-active, `−5` on a conflicting stack, recommended at ≥3 and capped at 4. No match means the always-on skills plus an explicit note, never a padded list.

### `tasks.md` — the shared status board

Every teammate edits this file, and only its own row.

```markdown
# GH-412 Partial Refunds — Task Tracker

**Decomposition strategy:** by architectural layer, schema first
**Total phases:** 6 across 4 rounds
**Team name:** `gh-412-partial-refunds`

## Phase status

| # | Phase | Round | Owner agent | Status | Agent | Started | Finished | Commits | PR |
|---|---|---|---|---|---|---|---|---|---|
| 00 | Flyway migration + refunds table | 0 | sql-pro | pending | — | — | — | — | — |
| 01 | RefundService partial-refund domain | 1 | java-pro | pending | — | — | — | — | — |
| 02 | Reconciler: refunds are non-terminal | 1 | python-pro | pending | — | — | — | — | — |
| 03 | Helm values + ConfigMap for the flag | 1 | kubernetes-architect | pending | — | — | — | — | — |
| 04 | POST /payments/{id}/refunds | 2 | java-pro | pending | — | — | — | — | — |
| 05 | E2E: Testcontainers + image + helm template | 3 | deployment-engineer | pending | — | — | — | — | — |

**Status legend:** `pending` · `in_progress` · `blocked` · `completed` · `dropped`

## File-conflict matrix (parallel rounds)

Round 1 file-conflict check — confirm before dispatch:

| File | Phase 01 | Phase 02 | Phase 03 |
|---|---|---|---|
| `payments-api/src/main/java/com/acme/payments/refund/RefundService.java` | Modify | — | — |
| `payments-api/src/main/resources/application.yml` | Modify | — | Modify |
| `reconciler/src/reconciler/handlers.py` | — | Modify | — |
| `deploy/helm/payments/values.yaml` | — | — | Modify |
| `deploy/helm/payments/templates/configmap.yaml` | — | — | Modify |

**Coordination rule — `application.yml`.** Phase 01 owns this file and adds
`payments.partial-refunds.enabled: false`. Phase 03 must NOT edit it; it templates
the key into the ConfigMap and documents the property name in its notes. If Phase 03
finds the property missing, it reports a blocker rather than adding it.

## Coordination Notes

- **Inferred:** the master plan does not say whether the reconciler's checkpoint
  survives the schema change. Phase 02 assumes it does and verifies with a restart
  test. Flagged here because it was inferred, not specified.
```

That last block is the one to read before you dispatch anything. **The `## Coordination Notes` section flags what the skill had to infer because the master plan did not say.** Every flag there is a place a wrong assumption can propagate into three parallel agents.

### A phase file, in full shape

`phases/PHASE-01-refund-service-domain.md`, abbreviated at the repetitive parts:

```markdown
# GH-412 / Phase 01 — RefundService partial-refund domain

> **For agentic workers:** REQUIRED SUB-SKILL: invoke
> `superpowers:subagent-driven-development` before touching code.

**Goal:** RefundService accepts repeated partial refunds and rejects any that would
breach the captured total.

**Architecture:** Implements § Service layer of the master plan. Sits between the
existing `PaymentRepository` and the not-yet-existing REST endpoint of Phase 04.

**Tech Stack:** Java 21, Gradle. Build `./gradlew :payments-api:build`.
Test `./gradlew :payments-api:test --tests "com.acme.payments.refund.*"`.
JUnit 5 + AssertJ + Mockito. Rules: `.editorconfig`,
`config/checkstyle/checkstyle.xml`, `.claude/rules/java.md`.

## Files

- **Create:** `payments-api/src/main/java/com/acme/payments/refund/PartialRefundPolicy.java`
- **Create:** `payments-api/src/test/java/com/acme/payments/refund/RefundServiceTest.java`
- **Modify:** `payments-api/src/main/java/com/acme/payments/refund/RefundService.java` — accept an amount, drop the terminal-state assumption
- **Modify:** `payments-api/src/main/resources/application.yml` — add the feature flag, default false

## Dependencies

- Phase 00: Flyway migration + refunds table

## Owner Agent

`java-pro`

## Risk / Effort

Risk: Medium — touches the payment state machine. Effort: ~2 h.

## Skills to Invoke (teammate-side)

1. `Skill(skill="superpowers:using-superpowers")` — establish skill discipline
2. `Skill(skill="superpowers:subagent-driven-development")` — execution discipline
3. `Skill(skill="superpowers:test-driven-development")` — red-green-refactor
4. `Skill(skill="superpowers:verification-before-completion")` — the done gate
5. `Skill(skill="database-design:postgresql")` — *the sum constraint is enforced in the schema too*

## Documents to Read

- `docs/architecture/payments-state-machine.md` — the CAPTURED → REFUNDED transition this phase makes non-terminal
- `docs/plans/GH-412/issue.specs` § 4 — GH-388 added the idempotency filter Phase 04 will rely on
- `.claude/rules/java.md` — the project's error-handling and logging conventions

## Pre-execution check

- [ ] **Step 01.0: Claim the phase.** Open `../tasks.md`, set Phase 01 →
      `Status = in_progress`, `Agent = phase-01`, `Started = YYYY-MM-DD HH:MM`.

## Atomic steps

- [ ] **Step 01.1: Locate the surface area.**

      codegraph explore "RefundService PaymentRepository — how does a refund flow today?"

      One call returns the symbols' line-numbered source, the call paths between
      them including dynamic-dispatch hops, and the blast radius. Fallback when the
      repo is not indexed: `rg --files -g '*.java' | rg 'RefundService\.java$'`.

- [ ] **Step 01.2: Author the first failing test** —
      `RefundServiceTest.rejects_refund_exceeding_captured_total`.
- [ ] **Step 01.3: Run it; expect FAIL** —
      `./gradlew :payments-api:test --tests "*RefundServiceTest*"`.
- [ ] **Step 01.4: Implement the minimum to pass.**
- [ ] **Step 01.5: Run it; expect PASS.**
- [ ] **Step 01.6+: Remaining tests** — exact-total refund closes the payment,
      zero and negative amounts rejected, concurrent refunds serialised.
- [ ] **Step 01.k-3: Full gate** — `codegraph affected <changed files>` to pick the
      scoped run first, then `./gradlew :payments-api:build` and `./gradlew spotlessCheck check`.
- [ ] **Step 01.k-1: TDD proof.** Mutate the policy to always-allow, confirm the
      over-refund tests DO fail, restore.
- [ ] **Step 01.k: Mark complete** — `Status = completed`, `Commits = (pending batch)`.
      **Do NOT request a commit.**

## Verification

- [ ] Repeated partial refunds succeed until the captured total is reached
- [ ] The refund that would breach the total returns a domain error, not an exception
- [ ] `./gradlew :payments-api:build` → 0 warnings, 0 errors; `spotlessCheck` clean
- [ ] TDD-proof step performed and described in the tasks.md notes

## Notes for downstream phases

- Phase 04 must pass the idempotency key from `IdempotencyKeyFilter` into
  `RefundService.refund(...)`; the service does not read it from the request context.
- The property `payments.partial-refunds.enabled` lives in `application.yml` and is
  owned by THIS phase. Phase 03 templates it into the ConfigMap and must not edit
  the YAML directly.
```

Two details in there carry most of the value. **The TDD proof** — deliberately break the implementation, confirm the tests that should fail do fail, restore — is what separates tests that verify behaviour from tests that merely execute code. And **Notes for downstream phases** is how a phase talks to a phase that has not started yet, in a run where they never share a conversation.

### Read the output before you run anything

| File | What to check |
|---|---|
| `phases/PHASE-NN-*.md` | Are these phases you would have written yourself? Is the Files list complete, or is a registration file missing? |
| `tasks.md` | Read every flag in `## Coordination Notes`. Those are the inferences. |
| `execute-plan.md` | The round list, then the Coordinator Prompt you paste. |
| `handoff.md` | An empty scaffold; the coordinator fills it after the last round. |

**Vague or overlapping phases mean an underspecified master plan.** Do not patch the phases — go back and sharpen the plan. Patching phases fixes the symptom in one place and leaves it everywhere else.

---

# Part 2.5 · Review the plan before you build it — optional

This is the cheap review, and the one people skip. A wrong phase boundary caught here costs minutes; the same boundary caught in round 2 costs a round and a confused human.

```
/plan-review-prompt
```

It asks for the planning folder and what to review — `master plan`, `phases`, or `all` — then reads the plan, harvests every path it cites, **verifies each of those paths actually exists**, and assembles a grouped, annotated reading list. A cited path that does not exist is kept and flagged, because a dangling reference is itself a finding.

It prints the prompt, offers to save it, and then offers to run it. Both offers are opt-in:

```sh
codex exec --sandbox read-only ${CODEX_OPENROUTER_MODEL:+-m "$CODEX_OPENROUTER_MODEL"} \
  -o docs/plans/GH-412/codex-review-all.md \
  < docs/plans/GH-412/codex-review-prompt-all.md
```

`--sandbox read-only` is the point — a review must not touch what it judges — and `-o` captures the report verbatim, so you read the reviewer's own words.

**On an indexed repo, CodeGraph is used on both sides of this handoff.** The skill builds the reading list from the graph — `codegraph explore` for the surface the plan touches, `codegraph impact` for the caller the plan never mentioned — and the prompt then carries a Tooling block telling the reviewer to verify the same way. The order matters: a reading list harvested only from what the plan *cites* inherits the plan's blind spots, so the review would be thorough about everything except the thing the plan forgot.

**The two gates, and who does what.** This shape is identical for both review skills — only the inputs differ:

```mermaid
sequenceDiagram
    autonumber
    participant U as You
    participant S as the review skill · Claude
    participant F as the plan folder
    participant X as codex exec · separate session
    participant R as the repo — read-only

    U->>S: /plan-review-prompt  (or /plan-implementation-review)
    S->>U: which folder? what am I reviewing?
    U-->>S: docs/plans/GH-412 · all
    S->>F: read the plan / the phases / tasks.md
    S->>R: harvest every cited path — and verify it exists
    R-->>S: real paths, plus one that is missing → flagged as a finding
    S->>R: codegraph impact / affected — the blast radius the plan never cited
    R-->>S: callers reached by dynamic dispatch + the tests they touch
    S->>U: the prompt, printed inline

    rect rgb(240, 240, 240)
        Note over S,U: gate 1 — opt-in
        S->>U: save it to codex-review-prompt-all.md?
        U-->>S: yes
        S->>F: write the prompt file
    end

    rect rgb(240, 240, 240)
        Note over S,U: gate 2 — opt-in, never automatic
        S->>U: run it now?
        U-->>S: yes
        S->>X: codex exec --sandbox read-only -o report.md, prompt on stdin
    end

    X->>R: git status · git diff · read files · codegraph explore
    R-->>X: ground truth, regenerated live
    X->>F: write the report, verbatim
    X-->>S: done
    S->>U: the report is at codex-review-all.md — read it there

    Note over S,U: the skill never reviews, never summarises the report<br/>in its place, and never acts on findings unprompted
```

Decline either gate and you still have the prompt, which is the deliverable — any reviewer with read access to the repo will take it.

A decomposition review of GH-412 would be looking for exactly the kind of thing this plan almost got wrong: whether the `application.yml` overlap is genuinely resolved or merely noted, whether Phase 02 can really run before the API exists, and whether Phase 05's verification names commands that exist.

---

# Part 3 · Execute — in a fresh conversation

`execute-plan.md` has three parts and **only the middle one is for the machine**: a round list for you at the top, one fenced ~170-line block under `## Coordinator Prompt`, and tips for you at the bottom. Paste only the fenced block, as your very first message, with no preamble.

````bash
awk '/^## Coordinator Prompt/{f=1;next} f&&/^```/{c++;next} f&&c==1' \
  docs/plans/GH-412/execute-plan.md | clip.exe      # pbcopy on macOS, xclip -sel c on Linux
````

Then the coordinator loops. Round 1 of GH-412 looks like this:

```mermaid
sequenceDiagram
    participant U as You
    participant C as Coordinator
    participant T as tasks.md
    participant P1 as phase-01 · java-pro
    participant P2 as phase-02 · python-pro
    participant P3 as phase-03 · kubernetes-architect

    U->>C: paste the Coordinator Prompt
    C->>T: read master plan + tasks.md + every phase file
    C->>U: summary + file-conflict verdict

    Note over C,P3: Round 1 — all three dispatched in ONE message
    C->>P1: Agent(subagent_type java-pro, name phase-01)
    C->>P2: Agent(subagent_type python-pro, name phase-02)
    C->>P3: Agent(subagent_type kubernetes-architect, name phase-03)

    P1->>T: status in_progress
    P2->>T: status in_progress
    P3->>T: status in_progress

    P3->>T: completed
    P2->>T: completed
    P1->>T: completed + deviation noted

    C->>C: full build + test gate
    C->>T: Round 1 summary in Coordination Notes
    C->>U: batched commit command — copy-pasteable, non-blocking
    Note over C: advances to Round 2 without waiting
```

Four things about that diagram are load-bearing:

- **One message, three `Agent` calls.** Sequential dispatch forfeits the entire point of rounds. If you see the coordinator dispatching one at a time, stop it.
- **`tasks.md` is the only shared state.** The three teammates never talk to each other. Everything one needs from another was written into "Notes for downstream phases" at decomposition time.
- **Phases never commit.** Each leaves a clean, tested tree and writes `(pending batch)`. Parallel agents committing independently produce interleaved partial commits nobody can review, and per-phase approval stalls the run on every finish.
- **The commit command does not block.** The coordinator surfaces it and moves on; you run it whenever you like.

You can check in on a running teammate with `SendMessage(to="phase-01", message="summarise progress and update tasks.md")`.

### What a deviation looks like when it works

Phase 01 discovers that `RefundService` already takes an amount parameter, added by GH-388 and never used. It notes the deviation in its `tasks.md` progress section instead of silently adapting. The coordinator reads that at end-of-round and carries it into the Round 1 summary — so Phase 04, dispatched later in a different context window, is told the parameter already exists rather than adding a second one.

That is the whole mechanism: **deviations travel through the file, not through anyone's memory.**

### And when a phase blocks

Phase 02 finds that the reconciler's checkpoint format changed under it — the thing `tasks.md` flagged as *inferred, not specified*, back at decomposition. It does not guess:

```mermaid
sequenceDiagram
    autonumber
    participant C as Coordinator
    participant P2 as phase-02 · python-pro
    participant T as tasks.md
    participant U as You

    C->>P2: dispatch phase 02
    P2->>T: status in_progress
    P2->>P2: the checkpoint assumption does not hold
    P2->>T: status blocked + entry in Active blockers
    P2-->>C: one paragraph — what blocked, what it needs

    C->>T: read the blocker and the phase's progress notes
    alt the coordinator can resolve it
        C->>T: record the call under Decisions
        C->>P2: re-dispatch a fresh teammate, same name, blocker named in the prompt
    else it needs a human
        C->>U: pause the round with the blocker stated
        U-->>C: the answer
    end

    Note over C,T: rounds 2 and 3 do NOT start on a broken round 1 —<br/>the coordinator holds until the round is green
```

Two rules make this safe rather than chaotic: a blocked phase writes `Status = blocked` and fills **Active blockers** instead of improvising, and the coordinator records the resolution under **Decisions**. Six months later, `tasks.md` still explains why the reconciler works the way it does.

---

# Part 4 · Review what actually landed

Two reviews, answering different questions. Neither replaces the other.

```mermaid
flowchart LR
    D["the changeset"]
    D --> CR["/code-review<br/><b>Is this diff good?</b><br/>same session · can --fix or --comment"]
    D --> PIR["/plan-implementation-review<br/><b>Did we build what the plan said?</b><br/>separate session · needs the plan"]
    CR --> F["fixes"]
    PIR --> F
```

`/code-review` judges the diff on its own merits and can apply findings with `--fix` or post them with `--comment`. It cannot judge plan conformance, because judging that requires the plan.

```
/plan-implementation-review
```

It asks for the planning folder and what the diff is measured against — `HEAD` for pending changes, or `master` when your agents cannot commit and the work is already on a branch. **If the working tree is clean it stops and asks** rather than emitting a prompt over an empty diff.

Then it builds a prompt from two sources of truth at once: the plan (what was *promised*) and the changeset (what *landed*), plus the exact commands to reproduce the diff live, the untracked files listed separately because they carry no diff, a changed-file → plan-item map that is **navigational and never evaluative**, and your repository's own standards discovered rather than assumed. On an indexed repo the blast-radius half of that reading list comes from `codegraph impact` and the test list from `codegraph affected`, rather than being inferred from the hunks — a diff shows what changed, never who depended on it. Run it the same way:

```sh
codex exec --sandbox read-only ${CODEX_OPENROUTER_MODEL:+-m "$CODEX_OPENROUTER_MODEL"} \
  -o docs/plans/GH-412/codex-implementation-review.md \
  < docs/plans/GH-412/codex-implementation-review-prompt.md
```

Leave the tree alone while it runs — the reviewer regenerates the diff live, so edits mid-run produce findings against code that no longer exists.

### What it catches that a diff review cannot

| Failure | Why `/code-review` misses it |
|---|---|
| Phase 02 implemented half of what its phase file specified and marked itself complete | The code that *is* there is clean and tested |
| Phase 03 solved the flag differently — hardcoded in the chart instead of templated | It reads as a deliberate, tidy choice |
| The `application.yml` coordination rule was violated: both phases edited it, last write won | The diff shows one coherent file |
| A phase was skipped and nobody noticed | There is no diff to look at |

The status-honesty check is the one worth the run on its own: the reviewer compares what `tasks.md` *claims* is complete against what the code shows. In a run where six agents each report their own status, that is the only independent check that exists.

### Read the findings like findings

**They are claims, not verdicts.** Some are wrong. Verify each against the code before changing anything — accepting a wrong finding costs more than missing a marginal one. The generated prompt is what makes them checkable: it forces `file:line` citations, separates conformance facts from quality findings from preferences, and requires "I couldn't verify X" instead of an assumption.

---

# Part 5 · Close out

After the last round:

1. **Real commit SHAs and the final summary in `tasks.md`** — replace every `(pending batch)`.
2. **Every section of `handoff.md` filled**, especially **deviations from the plan**. That is where reviewers spend their attention and where unexamined assumptions hide.
3. **Anything durable folded into `.claude/rules/`** rather than left in a plan folder nobody reopens. If this run taught you that the team wants domain errors instead of exceptions, that belongs in a rule file, not in `docs/plans/GH-412/`.

```markdown
## Deviations from the plan

- Phase 01: `RefundService` already accepted an amount parameter, added unused by
  GH-388. Reused it instead of adding a second. Phase 04 was told at end-of-round.
- Phase 03: the flag is templated into the ConfigMap as planned, but the Helm value
  is named `partialRefunds.enabled`, not `payments.partial-refunds.enabled` — Helm
  values are camelCase by chart convention. The Java property name is unchanged.
- Phase 05: `helm template` smoke test added to CI, which the plan did not ask for.
  Cheap, and it would have caught the naming mismatch above.
```

---

## Command cheat sheet

````sh
# once per machine
/plugin marketplace add necofx/necofx-claude-marketplace
/plugin install create-master-plan@necofx
/plugin install decompose-plan@necofx
/plugin install plan-review@necofx
/plugin marketplace add anthropics/claude-plugins-official
/plugin install superpowers@claude-plugins-official
/plugin marketplace add wshobson/agents
/plugin install jvm-languages@claude-code-workflows      # + the bundles your stack needs
codegraph init                                            # optional, in the repo root

# per ticket
/create-master-plan 412                                   # → issue.specs, master-plan.md
/decompose-plan docs/plans/GH-412                         # → phases/, tasks.md, execute-plan.md
/plan-review-prompt                                       # optional — review the plan first
awk '/^## Coordinator Prompt/{f=1;next} f&&/^```/{c++;next} f&&c==1' \
  docs/plans/GH-412/execute-plan.md                       # paste into a FRESH conversation
/code-review                                              # the diff on its own merits
/plan-implementation-review                               # optional — the diff against the plan

# updates
/plugin marketplace update necofx
/plugin update plan-review@necofx
````

## The five checkpoints, in one place

| After | Check | If it is wrong |
|---|---|---|
| step 1 | The outline is at chunk level; validation gates name your real commands | Fix the plan now — it is the cheapest place |
| step 2 | The phases are ones you would have written; read every `## Coordination Notes` flag | Sharpen the plan, not the phases |
| step 2.5 | The external reviewer's findings on the plan | Fold them back into the plan before executing |
| step 3 | The coordinator dispatched a round in **one** message | Stop it — sequential dispatch wastes the whole design |
| step 4/5 | Deviations are recorded, not absorbed | An unrecorded deviation is the one that bites in production |

## When it goes wrong

Each plugin's README carries its own troubleshooting table. The three failures that come from *joining* the steps rather than from any one of them:

| Symptom | Cause and fix |
|---|---|
| The coordinator loses track mid-run, or repeats a round | Step 3 was not started in a fresh conversation. Restart it in an empty one — `tasks.md` holds the state, so nothing is lost. |
| Two phases in one round hit the same file anyway | The matrix caught it and the coordination rule was ignored, or it missed the file because the master plan never named it. Move one phase to the next round. |
| Agents ignore your conventions | Check a phase's "Documents to Read" — an empty list means there was nothing to cite. Writing `.claude/rules/*.md` is the highest-leverage thing you can do here, and nobody can ship it for you. |
