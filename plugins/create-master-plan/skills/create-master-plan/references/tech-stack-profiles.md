# Tech-Stack Profiles

Reference profiles for the tech stacks this skill commonly encounters. When generating phase files, the orchestrator prompt, or the handoff document, pick the profile that matches the project's stack and substitute its fields into the templates.

## Detecting the project's stack

Use this precedence:

1. **Master plan declares it.** If the master plan has a `## Tech Stack` section, that's the authoritative answer.
2. **Root markers.** Look for these at or above the working directory — first match wins:
   - `*.sln` / `*.csproj` → **.NET** (or **Blazor** if any `.razor` files exist under the solution)
   - `package.json` with `"react"` / `"next"` / `"remix"` / `"gatsby"` in dependencies → **React**
   - `*.dproj` / `*.groupproj` / `*.dpr` → **Delphi**
   - `build.gradle` / `build.gradle.kts` / `settings.gradle(.kts)` / `gradlew` → **Java / JVM (Gradle)**
   - `pom.xml` / `mvnw` → **Java / JVM (Maven)**
   - `pyproject.toml` / `setup.py` → Python (consult user — not profiled here)
   - `Cargo.toml` → Rust (consult user — not profiled here)
3. **Per-phase files hint.** If the master plan's File Map points a phase at `*.razor` files, treat that phase as Blazor even when sibling phases are pure .NET.
4. **Ask the user** via `AskUserQuestion` when ambiguous or mixed.

For mixed projects (e.g. Blazor backend + React frontend), produce per-phase Tech Stack lines reflecting the layer each phase touches. Do not force a single profile on the whole plan.

---

## Profile: .NET

| Field | Value |
|---|---|
| Root markers | `*.sln`, `*.csproj` |
| Language | C# (typically 12+) |
| Build command | `dotnet build <solution>` |
| Test command | `dotnet test <solution>` |
| Scoped test | `dotnet test <project>.csproj --filter "FullyQualifiedName~<Name>"` |
| Test framework | xUnit (preferred), NUnit, MSTest |
| Assertions | AwesomeAssertions OR FluentAssertions (never mix in one project) |
| Mocking | NSubstitute (preferred) or Moq |
| Conventions location | `.editorconfig`, `.claude/rules/*.md`, `docs/development/` |
| Commit convention | `{TICKET}: <imperative>` is common; consult `git log` for the actual format |
| Common DI | Autofac, MS DI |
| Common ORM | NHibernate, Entity Framework Core |
| Common code-style flags | `var` allowed/forbidden, tabs vs spaces, brace style (Allman vs K&R), async-suffix policy |

**Where to find the project's actual rules:** read every `.claude/rules/*.md`, then `.editorconfig`, then any `STYLE.md` / `CONTRIBUTING.md`. Do NOT assume — different .NET shops have opposite preferences on `var`, tabs, and braces.

---

## Profile: Blazor

Inherits everything from **.NET**, plus:

| Field | Value |
|---|---|
| UI host | Blazor Server, Blazor WebAssembly (WASM), or Blazor Hybrid |
| Common component libraries | Telerik UI, MudBlazor, Radzen, Syncfusion, FluentUI |
| Component tests | bUnit + xUnit |
| E2E tests | Playwright (.NET binding) |
| Hot-reload | `dotnet watch run` — caveat: .razor changes hot-reload; some .cs changes don't |
| File mix in a phase | `.razor` (template), `.razor.cs` (code-behind), `.razor.css` (scoped styles) |
| Render mode considerations | Server / WASM / Auto changes behavior — phases must call out which one applies |

When a phase touches `.razor` files, the Verification step must include rendering the page in the dev server (not just `dotnet build`).

---

## Profile: React

| Field | Value |
|---|---|
| Root marker | `package.json` with `react` / `next` / `remix` / `gatsby` dep |
| Language | TypeScript (preferred), JavaScript (ES2020+) |
| Package manager | Detect by lockfile: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `package-lock.json` → npm, `bun.lockb` → bun |
| Build command | `<pm> run build` (or framework-specific: `next build`, `vite build`) |
| Test command | `<pm> run test` |
| Scoped test | `<pm> test -- <pattern>` (Vitest/Jest) or `playwright test <file>` |
| Test framework | Vitest (preferred for Vite-based), Jest (legacy), Playwright (E2E) |
| Component testing | `@testing-library/react` + `@testing-library/jest-dom` |
| Assertions | Built into the runner (`expect`) + dom matchers |
| Mocking | `vi.mock` / `jest.mock` for modules; **MSW** for network |
| State libs | Redux Toolkit, Zustand, Jotai, TanStack Query |
| Type checker | `tsc --noEmit` as a separate gate, or `<pm> run typecheck` |
| Lint / format | ESLint + Prettier; check `.eslintrc.*` and `.prettierrc.*` |
| Conventions location | `.eslintrc.*`, `.prettierrc.*`, `tsconfig.json`, `CONTRIBUTING.md` |
| Commit convention | Conventional Commits (`feat:`, `fix:`, `chore:`) is common; consult `git log` |
| Component test colocation | `*.test.tsx` next to component OR under `__tests__/` |
| Story-based UI dev | Storybook is common; phases adding a component often add a `.stories.tsx` too |

**Build + test + lint + type-check are FOUR separate gates** in React projects. Every phase's Verification step must list all four (or note which ones the project actually runs).

---

## Profile: Delphi

| Field | Value |
|---|---|
| Root markers | `*.dproj`, `*.groupproj`, `*.dpr` |
| Language | Object Pascal (Delphi 10.x – 13.x) |
| UI framework | VCL (Windows desktop) or FireMonkey/FMX (cross-platform) |
| Build command | `msbuild <project>.dproj /p:Config=Debug /p:Platform=Win32` (or `Win64`) |
| Scoped build | Same; specify a single `.dproj`, not the `.groupproj` |
| Test framework | DUnitX (preferred), DUnit (legacy) |
| Assertions | `Assert.*` from DUnitX |
| Mocking | Delphi-Mocks, Spring4D |
| Common component libs | DevExpress (cxGrid, dxLayoutControl, dxBarManager), TMS Components, FireDAC (data access), Indy (network) |
| Conventions location | `docs/coding-standards.md`, `.claude/rules/*.md`, project STYLE doc |
| Commit convention | Project-dependent; consult `git log` |
| Common code-style flags | `begin/end` always, PascalCase identifiers, no trailing whitespace, indent step (2 or 4 spaces; rarely tabs), `T`-prefix for classes, `I`-prefix for interfaces |
| Build output | `.exe` (VCL) or platform-specific bundle (FMX) — verification often runs the binary |

Delphi compiles per-project, not per-solution; phases must specify which `.dproj` they touch and rebuild only that one for fast feedback. The full `*.groupproj` build is the Verification gate.

---

## Profile: Java / JVM

Covers Gradle and Maven projects, including Kotlin / Groovy / Scala sources that build through the same toolchain. The two build tools differ in every command below — **determine which one the repo uses before writing any command into a phase.**

| Field | Value |
|---|---|
| Root markers | `build.gradle`, `build.gradle.kts`, `settings.gradle(.kts)`, `gradlew` → Gradle; `pom.xml`, `mvnw` → Maven |
| Language | Java (8 / 11 / 17 / 21 LTS — read the actual level from `sourceCompatibility` / `maven.compiler.release`, don't assume), or Kotlin / Groovy / Scala on the JVM |
| Build command | Gradle: `./gradlew build` — Maven: `./mvnw -B verify` |
| Scoped build | Gradle: `./gradlew :<module>:build` — Maven: `./mvnw -B -pl <module> -am verify` |
| Test command | Gradle: `./gradlew test` — Maven: `./mvnw -B test` |
| Scoped test | Gradle: `./gradlew :<module>:test --tests "com.acme.FooTest"` — Maven: `./mvnw -B -pl <module> test -Dtest=FooTest` |
| Test framework | JUnit 5 / Jupiter (preferred), JUnit 4 (legacy), TestNG, Spock (Groovy), Kotest (Kotlin) |
| Assertions | AssertJ (preferred), Hamcrest, Truth, or JUnit's built-in `Assertions.*` — never mix styles within one module |
| Mocking | Mockito (+ `mockito-junit-jupiter`), MockK (Kotlin), WireMock (HTTP boundaries), Testcontainers (real infra in integration tests) |
| Conventions location | `.editorconfig`, `config/checkstyle/checkstyle.xml`, Spotless / Checkstyle / PMD / SpotBugs / ErrorProne config inside the build file, `.claude/rules/*.md`, `CONTRIBUTING.md` |
| Commit convention | Conventional Commits is common; consult `git log` for the actual format |
| Common DI | Spring / Spring Boot, Dagger, Guice, CDI (Quarkus / Jakarta EE), Micronaut |
| Common ORM | Hibernate / JPA, Spring Data, jOOQ, MyBatis |
| Common code-style flags | Google Java Format vs. Checkstyle/Sun style, 2 vs 4 spaces, star-import policy, `final` on params/locals, Lombok allowed or banned, nullability annotation family (JSpecify / JetBrains / `javax.annotation`) |
| Build output | `.jar` / fat-jar / `.war`; run via `./gradlew bootRun`, `./mvnw spring-boot:run`, or `java -jar build/libs/<app>.jar` |

**Always use the wrapper** (`./gradlew`, `./mvnw`) when it exists — a system-installed `gradle` / `mvn` may be a different version and produce failures unrelated to the change. On Windows the wrappers are `gradlew.bat` / `mvnw.cmd`.

**Multi-module is the norm.** Both tools build a reactor of modules. A phase must name the module(s) it touches and run the scoped build/test for fast feedback; the full-reactor build is the Verification gate.

**Build, test, and static analysis are separate gates.** Many Java repos fail CI on Checkstyle / Spotless / PMD / SpotBugs / ErrorProne long before a test fails. Read the build file for which plugins are actually wired in and list each real gate in the phase's Verification step (e.g. `./gradlew spotlessCheck check`). Gradle's `build` normally already runs `check`, and `mvn verify` normally already runs the analysis phase — confirm this in the build file rather than assuming either way.

**Android is NOT this profile.** `build.gradle` plus an `AndroidManifest.xml` or the `com.android.application` plugin means an Android project: the build works the same way, but the UI layer, test tooling (Espresso, Robolectric), and lifecycle concerns are mobile ones. Tag those phases `mobile` and consult the user — Android is not profiled here.

---

## Cross-stack notes

- **Read-only git constraint.** Many teams forbid the agent from running git writes; check the project's `.claude/rules/git-workflow.md` (or equivalent CONTRIBUTING.md) before deciding whether teammates can commit themselves or must hand the command to the user.
- **Coding rules files.** When the project has `.claude/rules/*.md`, those are authoritative — list the relevant ones in each phase's "Documents to Read" section instead of restating the rules.
- **Mixed stacks.** A common pattern is .NET API + React SPA, or Delphi desktop + .NET REST server. Tag each phase with the stack of the files it touches; the file-conflict matrix is per-stack-disjoint by construction (different file extensions).
- **CI command discovery.** Look at `.github/workflows/*.yml`, `azure-pipelines.yml`, `Jenkinsfile`, or `Makefile` for the actual commands CI runs — those are the safest "build" and "test" commands to copy into phase files.
- **Code navigation (stack-independent).** When the repository has a `.codegraph/` directory, use `codegraph_explore` (MCP) or `codegraph explore` / `node` / `callers` / `impact` / `affected` (shell) instead of `Glob`/`Grep`/`Read` loops. One call returns the relevant symbols' verbatim line-numbered source, the call paths between them, and what depends on them — a large token saving over reading every file a text search matched, and it follows dynamic dispatch (callbacks, DI resolution, component rendering) that a text search cannot. `codegraph affected <changed-files>` names the test files a change reaches, which is how a phase picks its scoped test run. This works the same on every stack. No `.codegraph/` directory means the repo is not indexed: use the ordinary tools and do not index it yourself.

## When this file misses your stack

Add a new profile here rather than hardcoding it in `SKILL.md` or the templates. The profile only needs to fill the fields in the table format above; everything else falls through to the cross-stack notes.
