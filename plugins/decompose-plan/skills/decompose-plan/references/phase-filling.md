# Step 7 — Filling Skills to Invoke and the Atomic steps

_Moved out of `SKILL.md` so it survives context compaction: the skill reads this file from disk at the step that needs it._

#### Filling "Skills to Invoke" (uses the Step 6.5 inventory)

For each phase, derive these per-phase tags from the master plan + the phase's Files/Goal/Architecture:

- **Stack tag(s)** — from the tech-stack profile(s) chosen in Step 3.5 (e.g. `.net`, `blazor`, `react`, `delphi`). Multiple if the phase straddles stacks.
- **Layer tag(s)** — by inspecting the phase's Files list and Goal: e.g. `data`, `persistence`, `service`, `api`, `ui`, `test`, `infra`, `docs`.
- **Domain tag(s)** — by keyword in the phase's Goal + Architecture: e.g. `security`, `performance`, `database`, `network`, `ui`, `deploy`, `debug`, `architecture`.
- **Verb tag** — `implement` (most code phases), `fix` (bug-fix phases), `refactor` (cleanup phases), `review` (audit phases), `document` (doc phases).

Then build the Skills to Invoke list:

1. **Always-on superpowers (in this order):**
   - `superpowers:using-superpowers` — "establish skill discipline"
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
