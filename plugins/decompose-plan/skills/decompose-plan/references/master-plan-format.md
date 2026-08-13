# Expected Master Plan Format

This document describes the shape of a well-formed master plan that the `decompose-plan` skill expects as input. The format is modeled on plans that have driven real multi-agent runs. Keep your own best plan folder around as the in-house canonical example.

A master plan that does NOT include every section below is still usable — the skill infers missing sections from the prose and marks them as inferred in `tasks.md § Coordination Notes`. But the closer the input matches this shape, the cleaner and safer the decomposition.

## Required sections

### 1. H1 title

`# {PLAN_KEY} — {Short title}` (e.g. `# GH-412 — Master Plan: Iteration 02 (Server Readiness Gate + License Validation)`). `{PLAN_KEY}` is the ticket id `create-master-plan` derived — `GH-<number>` for the default GitHub source, a tracker key like `ACME-1234` otherwise.

The `{PLAN_KEY}` is used as a prefix in every phase file (`# GH-412 / Phase 02 — …`) and in commit messages (`GH-412: …`) — unless the repository's `git log` shows a different convention, which wins.

### 2. Required sub-skill notice (optional but recommended)

A one-line callout at the top stating which superpowers skill teammates should use to execute each phase. Example:

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement each phase task-by-task.

### 3. Goal

Numbered list of 1–3 concrete deliverables. Each item is one sentence describing user-visible behavior. No implementation details.

### 4. Architecture

One paragraph naming the **rounds** (sequential vs parallel) and the count of teammates per round. Example:

> Seven independent atomic phases delivered in three rounds:
> - Round 0 (sequential): Phase 0 — Foundation contracts.
> - Round 1 (parallel): Phases 1–5 — five teammates working in parallel, no shared files.
> - Round 2 (sequential): Phase 6 — Wiring + integration.
> - Round 3 (sequential): Phase 7 — Integration tests + manual verification.

### 5. Tech Stack

The frameworks, build commands, test commands, and coding-rules files every phase must obey. Take the matching profile from `tech-stack-profiles.md` and fill in the project-specific paths. Examples per stack — keep only the one(s) that apply, delete the others:

**.NET example:**

> - .NET 8/10, C# 12+
> - xUnit + AwesomeAssertions + NSubstitute
> - Autofac modules
> - Build: `dotnet build <solution>`
> - Test: `dotnet test <solution>`
> - Coding rules: `.editorconfig`, `.claude/rules/*.md` (if present), `docs/development/*.md`

**Blazor example:** (same as .NET, plus)

> - Blazor Server / WebAssembly / Hybrid (specify which)
> - Component library: Telerik UI / MudBlazor / Radzen / FluentUI (specify)
> - Component tests: bUnit + xUnit
> - E2E: Playwright (.NET)

**React example:**

> - TypeScript + Vite (or Next.js / Remix — specify)
> - Package manager: pnpm / npm / yarn / bun (detect by lockfile)
> - Build: `<pm> run build`
> - Test: `<pm> run test` (Vitest/Jest) + `<pm> run e2e` (Playwright)
> - Lint: `<pm> run lint` (ESLint + Prettier)
> - Type-check: `<pm> run typecheck` (`tsc --noEmit`)
> - State: Redux Toolkit / Zustand / Jotai / TanStack Query (specify)
> - Coding rules: `.eslintrc.*`, `.prettierrc.*`, `tsconfig.json`, `CONTRIBUTING.md`

**Delphi example:**

> - Object Pascal, Delphi 10.x / 11.x / 12.x / 13.x (specify)
> - VCL or FireMonkey/FMX (specify)
> - Test: DUnitX
> - Mocking: Delphi-Mocks / Spring4D
> - Build: `msbuild <project>.dproj /p:Config=Debug /p:Platform=Win32` (or `Win64`)
> - Common components: DevExpress / FireDAC / TMS / Indy (specify what's used)
> - Coding rules: project's coding-standards doc + `.claude/rules/*.md` (if present)

For mixed-stack plans, list one block per stack and label each phase in § 6 with the stack it targets.

### 6. Phase index

A table with these columns:

| # | Phase | File | Owner agent skill | Effort | Risk |
|---|---|---|---|---|---|
| 0 | Foundation contracts | `phases/PHASE-00-…md` | `dotnet-senior-developer` | ~30 min | Low |
| 1 | … | … | … | … | … |

The **Owner agent skill** column is what the decompose-plan skill writes into each phase file's `## Owner Agent` section. If absent, the skill defaults to `general-purpose`.

### 7. Parallelism graph (ASCII diagram)

A small ASCII flow showing rounds and parallelism. The decompose-plan skill copies this into `tasks.md § Rounds + dependencies`. Example:

```
Round 0 (sequential)
    Phase 0
        |
        v
Round 1 (parallel — 5 teammates)
    Phase 1   Phase 2   Phase 3   Phase 4   Phase 5
        |    (all five must complete before Round 2 starts)
        v
Round 2 (sequential)
    Phase 6
        |
        v
Round 3 (sequential)
    Phase 7
```

### 8. Wall-clock estimate

One line. Example: `~12 h sequential; ~7 h with Round 1 fully parallelised.`

### 9. Status tracking notice

A short paragraph stating that phases MUST update `tasks.md` at pickup / commit / completion. The decompose-plan skill writes the canonical update protocol into `tasks.md` itself, so this can be a short pointer.

### 10. File-structure map (CRITICAL for parallel safety)

An exhaustive table of every file each phase will **Create** or **Modify**, with phase ownership. Example:

| Action | File | Phase |
|---|---|---|
| Create | `src/.../IStartupPrerequisite.cs` | 0 |
| Create | `src/.../ServerReadinessGate.cs` | 1 |
| Modify | `src/.../BootstrapperModule.cs` | 5 |

The decompose-plan skill scans this map in Step 6 to build the **file-conflict matrix** for every parallel round. Two phases in the same round that touch the same file MUST have a documented coordination rule.

### 11. Pre-merge contract

A numbered list of hard requirements every phase's PR / commit must satisfy. Example:

1. Build clean: 0 warnings, 0 errors.
2. All tests pass.
3. Phase tests pass and fail without the fix (TDD discipline).
4. Commit message format: `{PLAN_KEY}: <imperative summary>`.
5. `tasks.md` updated.
6. Coding rules followed; no internal-context leaks in user-facing log strings.

The decompose-plan skill copies this verbatim into the phase template's `## Verification` section.

### 12. Out-of-scope (OOS)

A bulleted list of work explicitly NOT done in this plan. Example:

> - ❌ Heartbeat-timeout root cause investigation — separate ticket.
> - ❌ Continuous regression handling — out of scope per user decision.

OOS is as important as in-scope. The decompose-plan skill warns the user if any phase candidate it identified overlaps with the OOS list.

### 13. Verification at the end

A numbered list of end-to-end checks the coordinator runs after the last round completes. This becomes the basis of the operator manual-verification checklist (if there is one) and the `handoff.md § How to verify before merging` section.

### 14. Open risks / pre-execution checks

A numbered list of "things to verify before Round 0 starts" — assumptions in the plan that might not hold, contract uncertainties, lifetime concerns, version assumptions, etc. The coordinator should resolve these (or document the resolution) before dispatching Round 0.

## Optional sections

### Customer-facing release notes

User-visible wording of what changed, written in plain language, with a "Known limitation" subsection for acknowledged gaps. The decompose-plan skill doesn't generate this; the coordinator (or a downstream skill) may populate it after the last round.

### Operator manual-verification checklist

A numbered step-by-step the operator runs on a test bench to confirm the change works end-to-end. Distinct from the automated verification because the automated suite may have pre-existing flakiness.

### Coordination model

A short paragraph spelling out the coordinator/teammate interaction model. The execute-plan.md template the skill generates covers this; if the master plan also documents it, the two should agree.

## What "atomic phase" means

A phase is atomic when ALL of these hold:

- Single-focus outcome (no "and" in the goal).
- Independently verifiable (acceptance criteria runnable without other phases).
- Right-sized (30 min – 3 h wall-clock).
- Dependency-explicit (every dependency on another phase is named, not implied).
- Single owner agent (one `subagent_type` is responsible).
- File-disjoint within its round (no file-conflict with sibling phases — or with a documented coordination rule when unavoidable).

A phase that fails any of these is too coarse. Split it.

A phase with fewer than three meaningful atomic steps is too fine. Merge it with a neighbour.
