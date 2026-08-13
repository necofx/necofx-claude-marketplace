# master-plan.md — canonical skeleton

The `create-master-plan` skill writes this file in Step 8, AFTER the interview in Step 7 has filled the gaps in `issue.specs`. Apply the `superpowers:writing-plans` discipline on top of this skeleton, and substitute stack-specific values from `tech-stack-profiles.md`.

Sections are mandatory unless explicitly marked optional. Replace every `{placeholder}` with a real value. Replace every `_(none)_` placeholder with concrete content from the interview, or keep it if the interview confirmed there really is none.

For mixed-stack plans (e.g. .NET API + React UI), the Tech Stack section lists each stack with the layer it covers, and the layer-specific sections (Test Configuration, Validation & Testing) appear once per stack.

---

# {TICKET-ID} — Master Plan: {Short title}

## Context

This plan implements `{TICKET-ID}`. Sourced from:

- **Ticket:** {TICKET-ID} ({state}, opened by {author}, assigned to {assignee}) — source: {github | jira | linear | file | free-form}
- **Spec:** [`issue.specs`](issue.specs) in this folder
- **Attachments:** see [`attachments/`](attachments/) in this folder (or `_(no attachments)_`)
- **Date:** {YYYY-MM-DD}

{1 paragraph: the problem this plan solves, who asked for it, the user-visible outcome. No implementation details here.}

## Tech Stack

{One block per stack the work touches. For mixed-stack plans, list each. Pull values from `tech-stack-profiles.md`.}

- **{Stack name, e.g. .NET, Blazor, React, Delphi, Java/JVM}** — covers {layer this applies to}
  - Language / framework version
  - Build command
  - Test framework + assertion + mocking libs
  - Coding rules location

---

## Pre-flight Checklist

{Stack-aware checklist. Keep only the blocks for stacks actually in this plan; delete the others.}

**Always:**
- [ ] Read [`issue.specs`](issue.specs) in full
- [ ] Read every doc listed under § Related Docs below
- [ ] Confirm required MCP servers are connected (see § Implementation Outline)

**.NET / Blazor block (when applicable):**
- [ ] Read project's `CLAUDE.md` and `.claude/rules/*.md` (if present)
- [ ] Read project's coding standards under `docs/` (if present — common paths: `docs/development/DOTNET_CODE_STANDARDS.md`, `docs/development/DOTNET_CONVENTIONS.md`)
- [ ] Review `.editorconfig` for project-wide style rules
- [ ] For Blazor: confirm render mode (Server / WASM / Auto / Hybrid) and component library in use

**React block (when applicable):**
- [ ] Read `.eslintrc.*`, `.prettierrc.*`, `tsconfig.json`, `CONTRIBUTING.md`
- [ ] Confirm package manager + lockfile + Node version
- [ ] Review existing patterns in the affected directory (`src/components/`, `src/hooks/`, …)

**Delphi block (when applicable):**
- [ ] Read project's coding-standards doc + `.claude/rules/*.md` (if present)
- [ ] Confirm IDE version + target platforms (VCL Win32/Win64, FMX Windows/macOS/iOS/Android)
- [ ] Review affected units, forms, and packages

## Why

{1–3 sentences. The value/motivation: customer benefit, business driver, technical compliance, security ask. Sourced from interview "Goal & value".}

## Out of scope

Explicit non-goals — the boundary that prevents scope creep.

- {Item 1 — and a one-line reason it's excluded}
- {Item 2}
- _(none if everything in the interview was in-scope)_

---

## Technical Requirements

Grouped by layer when applicable. Each requirement is a complete sentence describing what must be true after the work lands. Pick the grouping that matches the stack(s); examples below — delete the ones that don't apply.

**.NET / Blazor grouping:**

### Data layer
- {Requirement}

### Persistence layer
- {Requirement}

### Service layer
- {Requirement}

### API layer
- {Requirement}

### UI / UX
- {Requirement (Blazor components — render mode, parameter shape, accessibility)}

**React grouping:**

### Components
- {Requirement (props contract, accessibility, error states)}

### Hooks / state
- {Requirement}

### API integration
- {Requirement (endpoint shape, error handling, retry policy, mocking)}

### Routing / pages
- {Requirement}

### Styling / theming
- {Requirement}

**Delphi grouping:**

### Domain units
- {Requirement}

### Services / business logic
- {Requirement}

### UI (forms / frames / components)
- {Requirement}

### Persistence / data access
- {Requirement}

**Cross-cutting (always):**

### Cross-cutting (logging, error-handling, security)
- {Requirement (project's log-hygiene rules, error-handling conventions, security review needs)}

---

## Implementation Outline

**Tech stack(s):** {pulled from § Tech Stack above; one line per stack for mixed projects}
**Primary owner-skill:** `{e.g. dotnet-senior-developer-skill / frontend-design / delphi-senior-developer-skill}`
**Specialised agents that may help:** {e.g. `code-reviewer`, `security-auditor`, `sql-pro`, `react-senior-developer`, `blazor-frontend-developer`, `performance-engineer`}
**MCP servers needed:** {e.g. `serena` for code search, `sequential-thinking` for complex reasoning, `context7` for library docs, `chrome-devtools` for React perf debugging. The default GitHub source needs none — it uses the `gh` CLI.}

**Sketch of the work** (atomic phase decomposition is left to `/decompose-plan`):

1. {High-level chunk 1 — 1 sentence describing the deliverable}
2. {High-level chunk 2}
3. {High-level chunk 3}
4. _(add more as needed; don't pre-decompose into phases)_

---

## Test Configuration

{One block per stack. Pull defaults from `tech-stack-profiles.md`; override with project-specific choices surfaced by the interview.}

**.NET / Blazor (when applicable):**
- **Framework:** xUnit (or NUnit / MSTest if the project uses them)
- **Assertions:** AwesomeAssertions (or FluentAssertions — never both in one project)
- **Mocking:** NSubstitute (preferred) or Moq
- **Test project:** {path}
- **Blazor component tests:** bUnit + xUnit (when UI is involved)
- **E2E:** Playwright (.NET) (when applicable)
- **Data strategy:** {synthetic / fixture / TestContainers / in-memory DB}

**React (when applicable):**
- **Unit / component framework:** Vitest (or Jest) + `@testing-library/react` + `@testing-library/jest-dom`
- **Network mocking:** MSW
- **E2E:** Playwright (or Cypress)
- **Test colocation:** `*.test.tsx` next to component OR under `__tests__/`
- **Data strategy:** {fixture, factory, MSW handlers}

**Delphi (when applicable):**
- **Framework:** DUnitX (or DUnit for legacy code)
- **Assertions:** `Assert.*` from DUnitX
- **Mocking:** Delphi-Mocks or Spring4D (or integration-style only — no mocks)
- **Test project:** {path to the test `.dproj`}

**Categories required (across all stacks):**
- Happy path
- Edge cases ({list specific ones from interview})
- Error paths ({…})
- Cancellation / abort ({…})
- Concurrent / race conditions ({…, when applicable})
- Accessibility ({Blazor / React UI phases})

---

## Validation & Testing

{Pick the block matching the stack(s). Combine for mixed-stack.}

**.NET / Blazor block:**
- [ ] `dotnet build <solution>` → 0 warnings, 0 errors
- [ ] All new and pre-existing tests pass (`dotnet test <solution>`)
- [ ] Scoped test run green: `dotnet test … --filter "…"`
- [ ] For Blazor UI work: launch dev server (`dotnet watch run`) and exercise change in browser
- [ ] Code review (human + `code-reviewer` agent) completed
- [ ] Operator manual-verification (if applicable) — see § Acceptance Criteria

**React block:**
- [ ] `<pm> run typecheck` → no TS errors
- [ ] `<pm> run lint` → clean (or only expected warnings)
- [ ] `<pm> run build` → builds successfully
- [ ] `<pm> run test` → all unit/component tests green
- [ ] `<pm> run e2e` → green (if E2E exists)
- [ ] Manual exercise in dev server (`<pm> run dev`)
- [ ] Accessibility scan clean (axe / Lighthouse)
- [ ] Code review (human + `code-reviewer` agent) completed

**Delphi block:**
- [ ] `msbuild <groupproj> /p:Config=Debug /p:Platform=Win32` → 0 errors
- [ ] DUnitX test exe(s) green
- [ ] Launch the VCL/FMX exe and exercise change manually
- [ ] Code review (human + `code-reviewer` agent) completed
- [ ] No memory leaks (FastMM4 / madExcept report clean, if configured)

---

## Acceptance Criteria

Checkbox list. Every item must be **testable** — observable in code, runtime behavior, log output, or operator console.

- [ ] {Criterion 1 — from the ticket's stated AC, verbatim}
- [ ] {Criterion 2 — from the ticket's stated AC}
- [ ] {Criterion 3 — surfaced by the interview}
- [ ] {…}

---

## Success ✅

All acceptance criteria met + every box in § Validation & Testing checked.

---

## Attachments

{List every file under `attachments/` with a one-line description of what it is and how it informs the plan.}

- `attachments/{file1.pdf}` — {one-line description}
- `attachments/{file2.png}` — {one-line description}
- _(none if no attachments on this ticket)_

---

## Related Docs

{Condensed from `issue.specs` § Related local docs — one line per file, with a sentence on why it's relevant to this plan. Same list, just enriched with relevance.}

- [`{relative-path}`](../../../{relative-path}) — {sentence on what to learn from it}
- [`{relative-path}`](../../../{relative-path}) — {sentence on what to learn from it}
- _(none if no related docs found)_

---

## Notes for the implementer

Anything that would be useful to a fresh agent picking this up cold but doesn't fit the categorised sections above. Examples:

- Naming conventions specific to this area of the codebase
- Existing utilities to reuse instead of re-implementing
- Stop-gaps in the current code to be aware of
- Performance concerns the team has voiced informally
- Browser support matrix (React) / IDE version constraint (Delphi) / target framework (.NET)

If `superpowers:writing-plans` was unavailable when this plan was generated, mention the fallback here. Otherwise, omit this section.
