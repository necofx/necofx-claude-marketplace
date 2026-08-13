# Interview Protocol (embedded into `create-master-plan`)

A project-agnostic, tech-stack-aware adaptation of the spec-interview pattern (inspired by the per-project `/interview` slash-command commonly found in `.claude/commands/interview.md`). This version is reusable across any project that has an `issue.specs` file produced by the parent skill.

## Where this protocol fits

- **Input:** `<plan-folder>/issue.specs` (written by Step 6 of the parent skill).
- **Output:** `<plan-folder>/master-plan.md` (written by Step 8 of the parent skill, after the interview ends).
- **Side effect:** the interview appends a `## Interview Notes` section at the bottom of `issue.specs` capturing every Q&A so future runs see what was discussed.

## Interview rules

- Use the `AskUserQuestion` tool for ALL questions. Never ask in free-form prose.
- Group related questions into a single `AskUserQuestion` call when possible (up to 4 questions per call).
- Each question must be **non-obvious** — skip anything `issue.specs` (Description, AC, Comments, Linked issues, Referenced documents, Interview Notes from a prior run) already answers.
- Ground questions in the code BEFORE asking; only ask the user about what cannot be inferred. When the repository has a `.codegraph/` directory, inspect the surface area with a single `codegraph_explore` call (or `codegraph explore "<symbols or question>"`) instead of a `Glob`/`Grep`/`Read` loop — one call returns the relevant symbols' source, the call paths between them, and what depends on them, at a small fraction of the tokens a read-every-match loop costs. Fall back to `Read`, `Glob`, `Grep` when the repo is not indexed.
- Continue until either: the user signals completion (e.g. "that's enough", "write the plan"), OR every box in the coverage matrix below is ticked.
- The first question is always **stack confirmation** when Step 5.5 of the parent skill found ambiguity (or when the project is multi-stack). Otherwise skip and proceed.

## Coverage matrix (use as a checklist; project-agnostic)

Tick each area only after at least one targeted question has been asked AND answered.

- [ ] **Stack confirmation** (skip if Step 5.5 was unambiguous) — which stack(s) does this work touch?
- [ ] **Goal & value** — why is this being built? What user-visible behavior changes?
- [ ] **Constraints** — what must stay true after the change (perf budget, backward compatibility, security, regulatory, deadlines, supported platforms / browser versions / Delphi version)?
- [ ] **Scope boundary** — what's explicitly in, what's explicitly out, what's "nice to have but not now"?
- [ ] **Technical implementation** — depends on the stack (see "Stack-specific question banks" below).
- [ ] **UI / UX** *(only if user-facing surface exists)* — flows, states, error messages, loading states, empty states, accessibility (keyboard nav, screen reader, color contrast).
- [ ] **Edge cases** — null / empty / max-size / concurrent / cancellation / network failure / partial failure / race conditions.
- [ ] **Tradeoffs** — what alternatives were considered, and why rejected?
- [ ] **Acceptance criteria** — concrete, testable, checkbox form. Match the ticket's stated AC if present; fill gaps.
- [ ] **Testing strategy** — unit / integration / e2e split, test framework + assertion lib + mocking lib (from the stack profile), data strategy, fixture vs synthetic.
- [ ] **Validation requirements** — build gate, lint gate, type-check gate, code-review gate, manual smoke test, operator verification.

## Stack-specific question banks

Pick the bank that matches the stack(s) resolved in Step 5.5. For mixed-stack projects, use multiple banks (one per layer).

### .NET / Blazor

- Which layer(s) does this touch — Domain / Persistence / Service / API / UI?
- DI lifetime for new services — singleton / scoped / transient? Any captive-dependency risks?
- Async patterns — does this code need `CancellationToken` propagation? `ConfigureAwait(false)`?
- ORM — NHibernate / EF Core / Dapper? Existing mapping convention to follow?
- Logging — `ILogger<T>` + structured templates? Project's log-hygiene rules?
- Exception handling — wrap, rethrow, swallow? Any specific exception types to catch?
- For Blazor: render mode (Server / WASM / Auto / Hybrid)? Component library (Telerik / MudBlazor / FluentUI)? State scope (single component / cascading parameter / state container)?

### React

- TypeScript or JavaScript? Strict mode on?
- Framework — bare React + Vite, Next.js (App Router vs Pages Router), Remix, Gatsby?
- State approach — local state, Context, Redux Toolkit, Zustand, Jotai, TanStack Query?
- Data fetching — fetch / axios / TanStack Query / RTK Query / framework loader?
- Forms — react-hook-form / Formik / native? Validation library — Zod, Yup, native?
- Styling — CSS Modules, Tailwind, styled-components, vanilla-extract, Emotion?
- Component library — shadcn/ui, MUI, Chakra, Mantine, Radix primitives, none?
- Testing — Vitest + Testing Library, Jest + Testing Library? Playwright for E2E?
- Network mocking — MSW (preferred), or runner-built-in mocks?
- Accessibility constraints — WCAG level the project targets?

### Delphi

- Delphi version (10.x / 11.x / 12.x / 13.x)?
- UI framework — VCL or FMX (and which platforms for FMX)?
- Data access — FireDAC / native ADO / dbExpress / something else?
- Component libraries used — DevExpress (cxGrid, dxLayoutControl), TMS, Indy, others?
- Test framework — DUnitX, DUnit?
- Mocking — Delphi-Mocks, Spring4D? Or no mocks (integration-style tests only)?
- Memory model — ARC interfaces or manual `TObject.Free`?
- Threading — anything async? `TThread`, `ITask`, OmniThreadLibrary?

### Mixed (e.g. .NET API + React frontend)

Run both banks. Add a coordination question: which layer ships first? Are API contracts already locked, or are they part of this work? Should TypeScript types be generated from the C# DTOs (e.g. via NSwag, TypeGen) or hand-written?

## Stack-agnostic question bank (always applicable)

- Branch convention — does the project require a specific branch naming pattern (e.g. `feature/{KEY}-…`, `fix/{KEY}-…`)?
- Commit convention — `{KEY}: <imperative>`, Conventional Commits (`feat:`, `fix:`), or none?
- Code-review process — solo merge, PR with one reviewer, two reviewers, CI required?
- CI signals — what does CI run? Is there a "must-be-green" gate or is it advisory?
- Release path — feature flag, environment, gradual rollout, big-bang?
- Telemetry — is there a metric / dashboard the team will watch after release?

## Before writing the master plan

1. Summarise the proposed master-plan outline in plain text (sections + one-line bullet each).
2. Ask the user to confirm via `AskUserQuestion` with options: **Looks good — write it**, **Edit the outline first**, **Ask more questions**.
3. Only proceed to Step 8 of the parent skill when the user picks "Looks good — write it".

## After every interview turn

Append a one-line summary of the Q&A to `<plan-folder>/issue.specs` under a `## Interview Notes` section at the very bottom of the file. Format:

```
- YYYY-MM-DD HH:MM — Q: <one-line question summary> · A: <one-line answer summary>
```

If `## Interview Notes` doesn't exist yet, create it. Never append elsewhere.

## Final master-plan structure (handed off to Step 8)

When Step 8 writes `master-plan.md`, ensure it includes ALL of these sections — shaped by `superpowers:writing-plans` and the matching stack profile from `tech-stack-profiles.md`:

### 1. Context
Brief paragraph: which ticket, why this plan exists, who asked for it. Link back to `issue.specs` and the `attachments/` folder.

### 2. Pre-flight Checklist
Stack-specific — built from the matching profile's "Conventions location" field plus the project's CLAUDE.md (if present). Example shapes:

- **.NET / Blazor:**
  - Read `issue.specs` in this folder
  - Read CLAUDE.md (if present) and relevant `.claude/rules/*.md` files (if present)
  - Read the project's coding standards doc(s) under `docs/`
  - Review the related local docs listed under § Related Docs
  - Identify affected components, layers, and DI registrations
  - Confirm required MCP servers are connected
- **React:**
  - Read `issue.specs` in this folder
  - Read `.eslintrc.*`, `.prettierrc.*`, `tsconfig.json`, `CONTRIBUTING.md`
  - Review existing patterns in the affected directory (`src/components/`, `src/hooks/`, …)
  - Confirm package manager + lockfile + Node version
  - Identify affected components, hooks, API routes, state slices
- **Delphi:**
  - Read `issue.specs` in this folder
  - Read the project's coding standards doc + `.claude/rules/*.md` (if present)
  - Identify affected units, forms, services, and packages
  - Confirm IDE version + target platforms

### 3. Why
1–3 sentences. The value/motivation: customer benefit, business driver, technical compliance, security ask. Sourced from interview "Goal & value".

### 4. Out of scope
Explicit non-goals — the boundary that prevents scope creep.

### 5. Technical Requirements
Concrete requirements discovered during the interview. Group by layer (data, persistence, service, API, UI for .NET/Blazor; or src/components, src/hooks, src/api, src/state for React; or domain/services/UI for Delphi) when applicable.

### 6. Implementation Outline

```markdown
## Implementation Outline

**Tech stack(s):** {pulled from Step 5.5; one line per stack for mixed projects}
**Primary owner-skill:** {e.g. dotnet-senior-developer-skill / frontend-design / delphi-senior-developer-skill}
**Specialised agents that may help:** {e.g. code-reviewer, security-auditor, sql-pro, performance-engineer}
**MCP servers needed:** {e.g. serena, sequential-thinking, context7 — the default GitHub source needs none}

**Sketch of the work** (the actual phase breakdown is left to /decompose-plan):

1. {High-level chunk 1}
2. {High-level chunk 2}
3. {High-level chunk 3}
```

Stop at the chunk level — do NOT pre-decompose into phases.

### 7. Test Configuration
Pulled from the matching stack profile:
- **.NET / Blazor:** xUnit + AwesomeAssertions + NSubstitute; bUnit for Blazor components; Playwright(.NET) for E2E
- **React:** Vitest or Jest + Testing Library + MSW; Playwright for E2E
- **Delphi:** DUnitX + Delphi-Mocks / Spring4D

Plus: test-project locations, data strategy, categories required (happy / edge / error / cancellation / concurrent / a11y for UI).

### 8. Validation & Testing
Checklist of gates the work must pass — pulled from the matching stack profile's build/test commands. Example shapes:

- **.NET / Blazor:** `dotnet build <solution>` → 0/0; `dotnet test <solution>` → all green; scoped tests; code-reviewer agent run.
- **React:** `<pm> run typecheck` clean; `<pm> run lint` clean; `<pm> run build` clean; `<pm> run test` green; `<pm> run e2e` green (if applicable); manual exercise in `<pm> run dev`.
- **Delphi:** `msbuild <groupproj> /p:Config=Debug` clean; DUnitX test exe(s) green; manual exercise of the binary.

### 9. Acceptance Criteria
Checkbox list extracted from the ticket's stated AC (if present) PLUS every additional criterion surfaced by the interview. Each criterion must be testable.

### 10. Success Criteria

```markdown
## Success ✅

All acceptance criteria met + Validation & Testing checklist complete.
```

### 11. Attachments
List every file under `attachments/` with a one-line description. If empty, write `_(no attachments on this ticket)_`.

### 12. Related Docs
Copy the list from `issue.specs` § Related local docs, condensed to one line per file with relevance.

## Output rules

- Write to `<plan-folder>/master-plan.md`. NEVER overwrite `issue.specs` from this step (the interview only appends to `## Interview Notes`).
- Use Markdown headings consistently. Code fences for code; tables for tabular data.
- Include specific details discovered during the interview (file paths, class names, prop signatures, test data examples). Vague master plans are useless to `/decompose-plan`.
- When the project enforces no-internal-context-in-log-strings (e.g. via `.claude/rules/logging.md`), call that out in Technical Requirements. Plan-level prose can name the ticket key freely; only recommendations for *code* must be hygienic.
