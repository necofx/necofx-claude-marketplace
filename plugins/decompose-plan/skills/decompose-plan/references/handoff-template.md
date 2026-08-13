# {PLAN_NAME} — Hand-off for code review

**Last touched:** {YYYY-MM-DD}
**Branch:** `{branch-name}`
**Status:** _(to be filled by the coordinator after the last round)_

_(This file is the single document a code reviewer reads first. The coordinator fills every section after the final round completes. Until then, the placeholders below stand in.)_

## What this iteration delivered

_(2–4 numbered bullets in plain English. Reference the master plan's goal. Call out the user-visible behavior change.)_

1. _(to be filled)_
2. _(to be filled)_

Out of scope this iteration (acknowledged): _(list what the master plan explicitly excluded; copy from the master plan's "Out-of-scope" section)_

## Background docs (read in this order before reviewing code)

1. `{MASTER_PLAN_FILENAME}` — architecture, phase index, file-structure map, pre-merge contract, OOS list.
2. `tasks.md` — phase-by-phase status board with per-phase deviations, test-project decisions, coordination flags.
3. `phases/PHASE-00-{slug}.md` … `phases/PHASE-{N}-{slug}.md` — per-phase atomic-step instructions, source of truth for the implementing teammates.

## Files touched

_(Quick summary by layer. The full map lives in the master plan's "File-structure map" section and the per-phase "Files" sections.)_

Group by whatever layering the project uses, e.g.:
- **.NET / Blazor projects** → Foundation / Domain / Persistence / BusinessLogic / API / UI / Tests
- **React projects** → src/components / src/hooks / src/api / src/state / src/pages / tests / e2e / styles
- **Delphi projects** → Source/Domain / Source/Services / Source/UI / Tests
- **Java / JVM projects** → group by reactor module first, then by package: `{module}/src/main/java/**/{domain,repository,service,controller}` / `{module}/src/test/java`

**Layer / area 1:**
- _(to be filled — file paths)_

**Layer / area 2:**
- _(to be filled)_

**Tests:**
- _(to be filled — list each new test file and the test count)_

Total: {N} new tests. Build status: {0 warnings, 0 errors / report deviations against the project's build command from the matching tech-stack profile}.

## Key deviations from the original plan (worth scrutinising)

_(Numbered list. For each: what the plan assumed, what the code does, and why. This is the section reviewers spend the most time on.)_

1. _(to be filled)_
2. _(to be filled)_

## TODOs / known limitations left in code

- _(every `// TODO: Track: {PLAN_KEY}` (or the project's equivalent) comment that landed, with rationale)_
- _(every acknowledged limitation; cite where it is documented in the master plan or release notes)_

## How to verify before merging

Substitute the project's actual commands from the matching tech-stack profile in `tech-stack-profiles.md`. Example shapes per stack:

**.NET / Blazor:**
1. `dotnet build <solution>` → must be 0 warnings, 0 errors.
2. Scoped tests:
   ```powershell
   dotnet test {test-project-1}.csproj --filter "{filter-1}"
   dotnet test {test-project-2}.csproj --filter "{filter-2}"
   ```
   Must all be green.
3. Full-solution `dotnet test <solution>` — _(note any pre-existing noise / known-flaky fixtures)._
4. For Blazor UI phases: launch the dev server (`dotnet watch run`) and exercise the change in the browser.

**React:**
1. `<pm> run typecheck` → no TS errors.
2. `<pm> run lint` → clean (or expected warnings only).
3. `<pm> run build` → builds successfully.
4. `<pm> run test` → all unit/component tests green.
5. `<pm> run e2e` (Playwright) → end-to-end green, if applicable.
6. Manually exercise the change in the dev server (`<pm> run dev`).

**Delphi:**
1. `msbuild <project>.groupproj /p:Config=Debug /p:Platform=Win32` → 0 errors.
2. Run the DUnitX test binaries built above → all green.
3. Launch the VCL/FMX exe and exercise the change manually.

**Java / JVM:** _(Gradle shown first, Maven second — keep only the one this repo uses, and always via the wrapper)_
1. `./gradlew spotlessCheck` / `./mvnw -B spotless:check` → formatting clean (skip if not wired in).
2. `./gradlew check` / `./mvnw -B verify` → compiles, and Checkstyle / PMD / SpotBugs / ErrorProne pass — these fail CI before any test does.
3. Scoped tests for the touched modules:
   ```bash
   ./gradlew :{module-1}:test --tests "{FQCN-1}"
   ./gradlew :{module-2}:test --tests "{FQCN-2}"
   ```
   Must all be green.
4. Full reactor: `./gradlew build` / `./mvnw -B verify` — _(note any pre-existing noise / known-flaky suites)._
5. Run the artifact and exercise the change (`./gradlew bootRun`, `./mvnw spring-boot:run`, or `java -jar {path-to-jar}`).

**Any stack:** the **operator manual-verification checklist** (always, if the master plan has one) — this is the ground-truth signal when the automated suite has pre-existing flakiness.

## Recommended code review

Use the `code-reviewer` agent or the `/code-review` skill against the diff of `{branch-name}` vs `{base-branch}`. Focus areas (typical signals per stack):

**.NET / Blazor:**
- Async patterns (`ConfigureAwait`, cancellation propagation, deadlock risks)
- DI lifetime correctness (no captive dependencies, scoped → singleton leaks)
- Logging hygiene (structured templates, no internal-context leaks in user-facing strings)
- Null/nullable handling, exception swallowing
- For Blazor: render mode correctness, re-render storms, lifecycle method usage, accessibility

**React:**
- Hook rules (deps arrays, conditional hooks, stale closures)
- Re-render storms (missing memoization, unstable references)
- Accessibility (a11y lint warnings, keyboard navigation, ARIA misuse)
- Error boundaries, suspense / loading states
- Type-safety at API boundaries (Zod / TS narrowing)
- Network mocking correctness in tests (MSW handlers cover the real shape)

**Java / JVM:**
- Null handling (Optional misuse, unchecked dereferences, nullability annotations at boundaries)
- Concurrency (shared mutable state, thread-pool sizing, `CompletableFuture` error propagation, blocking calls on reactive threads)
- Resource lifetime (try-with-resources, unclosed streams / connections, leaked executors)
- Spring specifics: bean scope correctness (singleton holding request state), `@Transactional` boundaries and self-invocation, N+1 from lazy JPA associations
- Exception discipline (swallowed exceptions, exceptions as control flow, lost causes when re-wrapping)
- Test quality (Mockito over-mocking, missing Testcontainers/integration coverage at real boundaries)

**Delphi:**
- Memory management (TObject lifetime, FreeAndNil discipline, interface ref-counting)
- Threading correctness (TThread, Synchronize/Queue)
- Exception handling (try/finally/except discipline)
- Database/transaction correctness (FireDAC scopes)
- UI event-handler signatures + form-leak prevention

**Cross-cutting (any stack):**
1. **{Component 1}** _(from Phase {NN})_ — _(what to look at, what specifically could go wrong)_
2. **{Component 2}** _(from Phase {NN})_ — _(what to look at)_

For each finding, suggest a concrete fix or document a follow-up. After the review, summarise verdict (ship / fix-before-merge / refactor-first) and list any new tickets to open.

## Open questions for the reviewer to consider

- _(to be filled — questions that deserve a second opinion but don't block ship)_
