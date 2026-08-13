# {PLAN_KEY} / Phase {NN} — {TITLE}

> **For agentic workers:** REQUIRED SUB-SKILL: invoke `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` before touching code. Atomic steps use checkbox (`- [ ]`) syntax for tracking — tick them off in this file as you go.

**Goal:** {One sentence describing the outcome of this phase. No "and"; if you need an "and", split the phase.}

**Architecture:** {2–3 sentences. What is being built, how it sits in the system. Reference the master plan section this implements.}

**Tech Stack:** {Pull from `tech-stack-profiles.md` for the stack this phase targets. Examples below — keep only the matching block, delete the others.}

<details>
<summary>Tech-stack examples (pick one)</summary>

- **.NET** — C# 12+, xUnit + AwesomeAssertions + NSubstitute. Build: `dotnet build <solution>`. Test: `dotnet test <solution>`. Coding rules: project's `.editorconfig` + `.claude/rules/*.md` (when present).
- **Blazor** — Same as .NET, plus Razor components, bUnit for component tests, Playwright (.NET) for E2E. Render mode: {Server / WASM / Auto / Hybrid}.
- **React** — TypeScript + Vite (or Next.js / Remix). Build: `<pm> run build`. Test: `<pm> run test` (Vitest/Jest) + `<pm> run e2e` (Playwright). Lint: `<pm> run lint`. Type-check: `<pm> run typecheck` (or `tsc --noEmit`). Mocks via `vi.mock` / `jest.mock` + MSW for network.
- **Delphi** — Object Pascal, VCL or FMX. Build: `msbuild <project>.dproj /p:Config=Debug /p:Platform=Win32`. Test: DUnitX. Mocks: Delphi-Mocks / Spring4D. Coding rules: project's coding-standards doc.
- **Java / JVM** — Java {17/21} (or Kotlin), {Gradle|Maven} — pick one, the commands differ. Build: `./gradlew :{module}:build` / `./mvnw -B -pl {module} -am verify`. Test: `./gradlew :{module}:test --tests "{FQCN}"` / `./mvnw -B -pl {module} test -Dtest={Class}`. JUnit 5 + AssertJ + Mockito. Static analysis: {Spotless/Checkstyle/PMD/SpotBugs — list only what the build file actually wires in}. Coding rules: `.editorconfig` + `config/checkstyle/checkstyle.xml`.

</details>

---

## Files

Exhaustive Create / Modify list. The coordinator uses this to detect file-conflicts across phases in the same round.

- **Create:** `path/to/new/file.ext`
- **Create:** `path/to/test/file.tests.ext`
- **Modify:** `path/to/existing/file.ext` — {one-line reason}

## Dependencies

{Phase IDs that must reach status `completed` before this phase can start. Use `None` for independent phases.}

- Phase {XX}: {Title}
- Phase {YY}: {Title}

## Owner Agent

{The agent type / skill that should execute this phase. Pick the right specialist for the tech stack. Examples by stack:}

- .NET: `dotnet-senior-developer` (and the `dotnet-senior-developer-skill`)
- Blazor: `dotnet-senior-developer` + `blazor-frontend-developer` (UI portions)
- React: `react-senior-developer`
- Delphi: `delphi-senior-developer` (and the `read-delphi-standards` skill)
- Java / JVM: `java-senior-developer` (or the project's JVM specialist — fall back to `general-purpose` when none is installed)
- Cross-cutting: `code-reviewer`, `security-auditor`, `sql-pro`, `performance-engineer`

Used as `subagent_type` by the coordinator. Default to `general-purpose` if no specialist fits.

## Risk / Effort

{Risk: Low / Medium / High. Effort: a wall-clock estimate, e.g. "~1 h", "~2 h", "30 min".}

---

## Skills to Invoke (teammate-side)

Invoke these skills via the `Skill` tool BEFORE doing any work. Order matters: always-on first, then matched.

This list is populated by `SKILL.md` Step 7 using the inventory built in Step 6.5 and the matching rules in `references/skill-matching-heuristics.md`. Every matched skill carries a 1-line "why" tied to THIS phase — never a parroted skill description.

**Always-on (every phase):**

1. `Skill(skill="superpowers:using-superpowers")` — establish skill discipline
2. `Skill(skill="superpowers:subagent-driven-development")` — execution discipline for this phase
3. `Skill(skill="superpowers:test-driven-development")` — red-green-refactor for the new tests *(omit for non-code phases)*
4. `Skill(skill="superpowers:verification-before-completion")` — required gate before marking complete

**Matched for this phase** (top 2–4 from the inventory, score ≥ 3 against the phase's tags):

5. `Skill(skill="<matched-name-1>")` — *<6–12 word "why", specific to this phase>*
6. `Skill(skill="<matched-name-2>")` — *<6–12 word "why">*
7. *(more if they pass the threshold, up to 4 total)*

Example shapes per stack (DELETE the example block once the real matched list is written):

- .NET: `Skill(skill="dotnet-senior-developer-skill") — implementing the new service registration in BootstrapperModule`
- Blazor: `Skill(skill="dotnet-senior-developer-skill") — wiring the new Razor component to the existing form state`
- React: `Skill(skill="react-senior-developer-skill") — implementing the new TanStack Query hook with cache invalidation`
- Delphi: `Skill(skill="delphi-senior-developer") — adding the new FireDAC unit and DUnitX tests`
- Cross-cutting: `Skill(skill="security-auditor") — reviewing the new JWT validation for OWASP issues`

When no domain-specific skills pass the score threshold for this phase, write only the always-on block and add this line:

> _(no domain-specific matches for this phase in the current skill inventory; always-on superpowers cover it)_

## Documents to Read

Exact files the teammate must read before executing tasks. Each line: path — one-sentence reason.

- `{docs-folder}/{section}/{file}.md` — {why this is relevant to this phase}
- Project coding rules (cite the actual files that exist; examples below — keep only those that apply):
  - `.claude/rules/*.md` — when the project keeps its coding rules as per-topic rule files
  - `CONTRIBUTING.md` / `STYLE.md` — common React convention
  - `.editorconfig` / `.eslintrc.*` / `.prettierrc.*` / `tsconfig.json` — React / TypeScript projects
  - project's coding-standards doc (often `docs/coding-standards.md`) — Delphi projects
  - `.editorconfig` / `config/checkstyle/checkstyle.xml` / the Spotless or Checkstyle block in `build.gradle(.kts)` / `pom.xml` — Java / JVM projects
- Stack-relevant rule files for the project (e.g. logging, error-handling, async, git-workflow when they exist as separate files)

If a file does not exist, report it back in the per-phase notes section of `tasks.md` and continue with what's available.

---

## Pre-execution check

- [ ] **Step {NN}.0: Claim the phase.** Open `../tasks.md`. Change Phase {NN} row → `Status = in_progress`, `Agent = phase-{NN}` (or your subagent name), `Started = YYYY-MM-DD HH:MM`. Append a "started — picked up" entry under your Detailed Progress section.

## Atomic steps

The TDD-discipline shape below applies when this phase delivers code. For research / ADR / doc-only phases, replace Steps {NN}.2–{NN}.5 with domain-appropriate atomic steps (one-line summary each) and keep the rest of the structure.

The command and code-snippet placeholders are intentionally stack-neutral. Substitute the matching profile from `tech-stack-profiles.md` (or the master plan's Tech Stack section) when writing the real phase.

- [ ] **Step {NN}.1: Locate the surface area being changed.**

	**When the repository has a `.codegraph/` directory, do this with ONE call and skip the search loop:**

	```bash
	# MCP:   codegraph_explore  query: "{Contract} {ComponentOrUnit} — how does it wire together?"
	# Shell: codegraph explore "{Contract} {ComponentOrUnit}"
	```

	One call returns the relevant symbols' verbatim line-numbered source grouped by file, the call paths between them (including dynamic-dispatch hops a text search cannot follow), and a blast-radius summary of what depends on them. That is what this step needs, and it costs a fraction of a glob-then-read-every-match loop — the search loop pulls in whole files to find a few signatures.

	Useful follow-ups, still one call each: `codegraph node {Symbol}` for one symbol's source plus its caller/callee trail, `codegraph callers {Symbol}` before changing a signature, `codegraph impact {Symbol}` to see what a change reaches.

	**Fallback when the repo is not indexed** (no `.codegraph/` — do not index it yourself unless the user asks):

	```bash
	# .NET / Blazor (PowerShell):
	#   Get-ChildItem -Path src -Recurse -Filter I{Contract}.cs
	# React (bash):
	#   rg --files src | rg "{ComponentName}\\.tsx?$"
	# Delphi (PowerShell):
	#   Get-ChildItem -Path Source -Recurse -Filter "*{ContractOrUnit}*.pas"
	# Java / JVM (bash):
	#   rg --files -g '*.java' -g '*.kt' | rg "{ClassName}\\.(java|kt)$"
	```

	Either way: read the existing signatures / props / interfaces and confirm them in the source. Note any deviation from what the master plan assumed (e.g. nullable vs non-nullable, namespace differences, missing props, default values). Surface deviations in the per-phase notes of `tasks.md`.

- [ ] **Step {NN}.2: Author the first failing test.**

	Path: `path/to/test/file.tests.{cs|tsx|pas}` (Java / JVM: `src/test/{java|kotlin}/{package}/{Class}Test.{java|kt}`)

	Write the smallest possible test that pins ONE behavior of the new code. Use the assertion + mocking libraries from the project's stack profile (NSubstitute + AwesomeAssertions for .NET; `expect` + Testing Library + MSW for React; DUnitX + Delphi-Mocks for Delphi).

- [ ] **Step {NN}.3: Run the new test; expect FAIL** (the type/component/unit does not exist yet).

	```bash
	# .NET / Blazor (scoped):
	#   dotnet test path/to/test.csproj --filter "FullyQualifiedName~{TestName}"
	# React (Vitest scoped):
	#   <pm> test -- {TestFile}
	# React (Playwright scoped):
	#   <pm> exec playwright test {TestFile}
	# Delphi:
	#   msbuild {TestProject}.dproj /p:Config=Debug && run the resulting test exe
	```

- [ ] **Step {NN}.4: Implement the minimum to make Step {NN}.2 pass.**

	Path: `path/to/new/file.{cs|tsx|pas}`

	Smallest possible production code that turns the failing test green. Follow the project's coding rules from "Documents to Read" — they govern style, async patterns, naming, error-handling, and logging.

- [ ] **Step {NN}.5: Run the test; expect PASS.**

- [ ] **Step {NN}.6+: Add the remaining tests** (edge cases, error paths, null/empty/cancelled/concurrent, parameter validation, accessibility for UI phases, contract types for TypeScript). Add one at a time; each fails first, then passes.

- [ ] **Step {NN}.k-3: Full build + test gate.**

	When the repo is indexed, find the test files your changes actually reach before running the full suite — a scoped run gives faster feedback, and it catches tests you did not know depended on this code:

	```bash
	codegraph affected {changed-file-1} {changed-file-2} --quiet
	```

	The full gate below still runs; `affected` only tells you what to run first.

	Substitute the project's build + test commands from the matching stack profile:

	```bash
	# .NET / Blazor:
	#   dotnet build <solution>
	#   dotnet test <solution>
	# React (run every gate the project actually has):
	#   <pm> run typecheck
	#   <pm> run lint
	#   <pm> run build
	#   <pm> run test
	# Delphi:
	#   msbuild <project>.groupproj /p:Config=Debug /p:Platform=Win32
	#   then run the DUnitX test exe(s)
	```

	Expected: zero warnings, zero errors, all tests green.

- [ ] **Step {NN}.k-2: Stack-specific verification.**

	If this phase touches UI, also verify in the browser / app shell (not just tests). For React/Blazor: launch the dev server and exercise the change. For Delphi: build and run the binary. The `superpowers:verification-before-completion` skill is mandatory here.

- [ ] **Step {NN}.k-1: TDD proof.** Temporarily mutate the production code to a trivially-wrong shape (e.g. early-return, hardcoded literal, no-op function body). Re-run the test filter from Step {NN}.3 and confirm the tests that should fail DO fail. Restore the real implementation.

- [ ] **Step {NN}.k: Mark phase complete.** Change Phase {NN} row in `tasks.md` → `Status = completed`, `Finished = YYYY-MM-DD HH:MM`, `Commits = (pending batch)`. Append a final summary entry under your Detailed Progress section: what was delivered, how many tests landed, any deviations from the plan.

	**Do NOT request a commit from the user.** Commits are deferred to an end-of-round or end-of-plan batch — the coordinator surfaces the batch commit command(s) once a coherent set of phases is green. Stay focused on delivering code that builds + tests cleanly; the commit happens later.

---

## Verification

Concrete, testable outcomes that prove the phase is complete. The coordinator and reviewer check these before flipping `Status = completed`.

- [ ] {Behavior 1 observable in code / tests / runtime}
- [ ] All new tests green; full stack-appropriate build is 0 warnings / 0 errors (per the profile's "Build command").
- [ ] **Stack-appropriate code-style check passes** — what counts varies by stack:
  - .NET / Blazor: project's `.editorconfig` + `.claude/rules/*.md` rules respected (e.g. `var` policy, brace style, async-suffix, structured logging templates, no internal-context leaks in user-facing log strings)
  - React: `<pm> run lint` clean, `<pm> run typecheck` clean, Prettier formatted
  - Delphi: project's coding-standards rules respected (begin/end blocks, PascalCase, no trailing whitespace)
- [ ] TDD-proof step performed and described in the per-phase notes.

## Notes for downstream phases

{Anything later phases need to know — registrations to add, surfaces to wire up, stop-gaps that must be removed, naming choices that need to stay consistent, prop contracts other components rely on. Be specific so the next teammate doesn't re-discover this from code.}

- {e.g. (.NET): "Phase {NN+2} must register `{NewType}.As<I{Contract}>().SingleInstance()` in `{BootstrapperModule}.cs`."}
- {e.g. (React): "The new `<{Component}>` exposes a `onSelect: (id: string) => void` prop — phases consuming it must pass a stable callback (memoize)."}
- {e.g. (Delphi): "The new `T{Class}` is registered in `{Module}.RegisterServices`; later phases must call `{ServiceLocator}.Resolve<I{Contract}>` to get it."}
- {e.g. (any): "Stop-gap at `{FilePath}:{LineHint}` — Phase {NN+1} must remove this when it lands the final flow."}
