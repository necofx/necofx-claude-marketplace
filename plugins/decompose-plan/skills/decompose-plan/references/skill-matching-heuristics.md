# Skill Matching Heuristics

Canonical rules used in Steps 6.5 + 7 of `SKILL.md` to match registered skills (from `.claude/skills/` and the runtime-active set) to each phase. Update the tables here when adding new tags; do NOT hardcode keyword lists inside `SKILL.md`.

## How matching works

1. Build the **inventory** in Step 6.5: every available skill tagged with `stack`, `domain`, `verb` tags derived from its frontmatter `description` using the keyword tables below.
2. For each phase, derive **phase tags** the same way from the phase's Goal + Architecture + Files list.
3. Score each candidate skill against the phase tags using the scoring rules below.
4. Recommend the top 2–4 with score ≥ threshold, plus the always-on superpowers.

## Keyword tables (case-insensitive, substring match)

### Stack tags

Apply to BOTH skill descriptions and phase content.

| Tag | Keywords |
|---|---|
| `.net` | `.net`, `dotnet`, `c#`, `csharp`, `nuget`, `nhibernate`, `entity framework`, `efcore`, `autofac`, `xunit`, `nunit`, `aspnet`, `asp.net`, `nsubstitute` |
| `blazor` | `blazor`, `razor`, `bunit`, `mudblazor`, `telerik`, `radzen`, `webassembly`, `wasm` (only when paired with `blazor` in same description) |
| `react` | `react`, `typescript`, `jsx`, `tsx`, `next.js`, `nextjs`, `remix`, `vite`, `vitest`, `playwright`, `testing library`, `redux`, `zustand`, `tanstack` |
| `delphi` | `delphi`, `object pascal`, `vcl`, `fmx`, `firemonkey`, `firedac`, `dunitx`, `dunit`, `devexpress`, `tms`, `spring4d` |
| `java` | `java` (**whole word only** — see the note below), `jvm`, `gradle`, `maven`, `spring boot`, `spring data`, `springframework`, `junit`, `jupiter`, `mockito`, `assertj`, `hibernate`, `jpa`, `jakarta`, `quarkus`, `micronaut`, `testcontainers`, `scala` (**whole word only**), `groovy`, `kotlin` (only when `android` does NOT also match — otherwise `mobile`) |
| `python` | `python`, `django`, `fastapi`, `flask`, `pytest`, `poetry`, `pip` |
| `sql` | `sql`, `postgres`, `postgresql`, `mariadb`, `mysql`, `sqlite`, `mssql`, `sql server`, `oracle`, `firebird`, `rdbms`, `t-sql`, `plsql` |
| `js` | `javascript`, `node`, `nodejs`, `npm`, `pnpm`, `yarn`, `bun` (use ONLY when `react` doesn't also match) |
| `mobile` | `ios`, `swift`, `swiftui`, `android`, `kotlin` (only when paired with `android` — otherwise `java`), `flutter`, `react native` |

A skill or phase may carry multiple stack tags. Mixed-stack projects are normal.

**Substring-matching pitfalls.** These tables match on substrings, which produces two false positives that must be guarded with a word boundary before the tag is applied:

- `java` is a substring of **`javascript`** — a React/Node skill would otherwise be tagged `java`, and the `−5` cross-stack penalty below would then fire on the wrong pairs. Tag `java` only on a whole-word match.
- `scala` is a substring of **`scalable`** / `scaling` / `escalate` — words that appear constantly in ordinary architecture prose. Same rule: whole word only.

`kotlin` is not a substring problem but a routing one: server-side Kotlin is `java`, Android Kotlin is `mobile`. Check whether `android` also matches before choosing.

### Domain tags

| Tag | Keywords |
|---|---|
| `security` | `security`, `auth`, `authentication`, `authorization`, `crypto`, `oauth`, `jwt`, `owasp`, `vulnerab`, `cert`, `pki` |
| `performance` | `performance`, `perf`, `benchmark`, `profil`, `optimiz`, `cache`, `latency`, `throughput`, `n+1`, `slow query` |
| `database` | `database`, `orm`, `migration`, `schema`, `index`, `query optimization`, `dba` |
| `network` | `network`, `snmp`, `tcp`, `udp`, `http`, `https`, `dns`, `routing`, `firewall`, `discovery` (when paired with network terms) |
| `ui` | `ui`, `ux`, `wireframe`, `design system`, `component`, `accessibility`, `a11y`, `responsive`, `figma` |
| `deploy` | `deploy`, `docker`, `container`, `kubernetes`, `k8s`, `helm`, `ci`, `cd`, `pipeline`, `github actions`, `jenkins`, `argo` |
| `debug` | `debug`, `diagnose`, `bug`, `troubleshoot`, `error`, `stack trace`, `incident`, `outage`, `production issue` |
| `architecture` | `architect`, `clean architecture`, `ddd`, `domain driven`, `adr`, `system design`, `microservice`, `service boundar` |
| `review` | `code review`, `audit`, `pr review`, `reviewer` |
| `test` | `test`, `tdd`, `integration test`, `e2e`, `coverage`, `mocking`, `fixture`, `red-green-refactor` |
| `legacy` | `legacy`, `modernize`, `modernization`, `migrate`, `refactor` (when paired with `legacy`) |
| `mcp` | `mcp`, `model context protocol`, `mcp server` |
| `documentation` | `document`, `runbook`, `readme`, `api docs`, `technical writing` |

### Verb tags

Pulled from the skill description's preamble (first ~50 chars) AND from the phase's Goal:

| Tag | Trigger words |
|---|---|
| `implement` | `implement`, `build`, `develop`, `create`, `add`, `introduce`, `scaffold` |
| `fix` | `fix`, `repair`, `resolve`, `correct`, `bug fix`, `hotfix` |
| `refactor` | `refactor`, `simplify`, `clean up`, `restructure`, `improve` (when not paired with `performance`) |
| `review` | `review`, `audit`, `inspect`, `validate` |
| `document` | `document`, `write docs`, `update readme`, `explain` |
| `test` | `test`, `verify`, `validate` (only when the goal is *adding* tests, not exercising existing ones) |

## Scoring rules (Step 7)

For each phase tag set `P = {stack, layer, domain, verb}` and each candidate skill's hints `S = {stack, domain, verb}`:

```
score = 3 * |P.stack ∩ S.stack|
      + 2 * |P.domain ∩ S.domain|
      + 1 * |P.verb ∩ S.verb|
```

Plus these adjustments:

- `+2` bonus if the skill's `source = active` (runtime-loadable). Project-local and user-global skills score normally but lose the bonus.
- `−5` penalty if the skill has a stack tag the phase does NOT have AND the phase has a stack tag the skill does NOT have. This avoids matching a `delphi-senior-developer` to a `react` phase just because both mention "test".
- `+1` bonus when the skill name literally contains a token from the phase title (last-resort tie-breaker — e.g. `dotnet-senior-developer` matches a phase titled "Build the .NET service module").

**Recommendation threshold:** include only skills with score ≥ 3 in the phase's "Skills to Invoke" list (beyond the always-on superpowers).

**Cap:** at most 4 additional skills per phase. If more pass the threshold, take the top 4 by score; tie-break by source (`active` > `project-local` > `user-global`), then by alphabetical name.

## Layer-tag derivation for phases

Layer comes from the phase's Files list and Goal, not from the inventory. Use these rules to tag a phase:

| Layer tag | Trigger (files or goal) |
|---|---|
| `data` | files under `Data/`, `Models/`, `Domain/`, `*.Data.Model/`, `domain/`, `model/`, `entity/`, `dto/`; goal mentions "entity", "model", "DTO" |
| `persistence` | files under `Persistence/`, `Repositories/`, `Mapping/`, `repository/`, `dao/`; mentions ORM, mapping, repository |
| `service` | files under `Services/`, `BusinessLogic/`, `service/`, `usecase/`; mentions "service", "use case" |
| `api` | files under `Controllers/`, `Api/`, `*.Api/`, `controller/`, `resource/`, `web/`; mentions REST, endpoint, route |
| `ui` | files under `Pages/`, `Components/`, `*.razor`, `*.tsx`, `*.jsx`, `Forms/`, `src/components/`; mentions component, page, screen, view |
| `test` | files under `Tests/`, `*.tests.*`, `__tests__/`, `*Tests/`, `src/test/java/`, `src/test/kotlin/`; mentions "tests", "TDD" |
| `infra` | files under `docker/`, `.github/`, `scripts/`, `deploy/`; mentions "pipeline", "deployment", "infrastructure" |
| `docs` | files under `docs/`, `README.md`, `*.md` only (no code); mentions "documentation", "ADR", "runbook" |

A phase may carry multiple layer tags (e.g. a phase that touches both `Services/` and `Tests/` gets `[service, test]`).

## When the inventory is sparse

If the runtime + project-local + user-global inventory turns up no domain matches for a phase (only the always-on superpowers apply), this is a SIGNAL, not a failure:

- Write the phase normally with only the always-on superpowers.
- Record a one-line note in `tasks.md § Coordination Notes`:
  `Phase {NN}: no matched domain/stack skills in the inventory — implementer relies on always-on superpowers only`
- This lets the user decide whether to install a missing specialist skill before dispatch, or accept the gap.

## When NOT to add a skill

Resist these tempting-but-wrong matches:

- A skill whose description merely *mentions* the stack (e.g. a Slack-GIF skill that says "create animations for engineering teams using React") — the stack must be the skill's actual domain.
- A skill that only matches on `verb` alone (e.g. `implement`) — verb is the weakest signal and should never single-handedly carry a recommendation.
- Two skills covering the same niche (e.g. `dotnet-senior-developer` and `dotnet-senior-developer-skill`) — pick one based on source + score.
- Generic skills like `general-purpose` — those go in the Owner Agent section, not Skills to Invoke.

## Update protocol

When adding a new stack or domain to the project:

1. Add the keywords to the table above.
2. Add a stack/domain profile to `tech-stack-profiles.md` if it's a new stack.
3. No `SKILL.md` change required — the workflow consults these tables at runtime.
