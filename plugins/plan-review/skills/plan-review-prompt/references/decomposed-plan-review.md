# Decomposed-plan review — instructions + output format (embeddable)

This file holds the two blocks the skill embeds into a generated Codex prompt when the review target is **phases / decomposed plan** (or as "Part 2" when the target is **all**).

A decomposition has different failure modes than a master plan: it can drop a requirement in the split, declare two phases "parallel" that actually share a file, or produce a phase a fresh teammate can't execute from the file alone. The checklist targets those.

**How to use:** copy the two fenced blocks below into the assembled prompt. Tailor the `<...>` placeholders. The decomposition's own structure (rounds, the file-conflict matrix, the dependency graph) gives you the high-judgment questions — lift the riskiest ones (a phase that bundles several units of work, a "parallel" round, a cross-phase dependency) into the explicit-questions list so the reviewer must rule on them.

**Tailoring notes:**
- Point the reviewer at the **master plan as the source of truth** the phases must cover — coverage/traceability is the single most important check.
- If a phase bundles multiple findings/units, or a round is marked parallel, name it in the high-judgment questions.
- Surface any "developer-dependent / blocked / manual" items the decomposition flagged, and have the reviewer confirm they're honestly marked outstanding rather than silently checked.

---

## When the target is "all"

Embed this under a "Part 2 — Decomposition review" heading after the master-plan instructions. The onboarding is shared (already assembled once). Add one combined verdict at the end (see the output format's note). Tell the reviewer: review the master plan first (Part 1), because the decomposition is judged against it.

---

## Block 1 — How to perform the review

```
# How to perform the review

Review the DECOMPOSITION — whether it faithfully covers the master plan,
whether its round/dependency/file-conflict structure is sound, and whether a
fresh teammate with zero prior context could execute each phase from its file
alone. Do not re-review the master plan's underlying technical decisions; do
flag any place a phase contradicts the master plan.

## 1. Coverage / traceability (most important)
- Build a matrix: every master-plan unit of work (each finding / requirement /
  acceptance criterion) → the phase that implements it. List any ORPHAN (a
  requirement no phase covers) and any INVENTION (a phase doing work not in the
  master plan).
- Confirm the easily-dropped items survived the split: cross-cutting
  compliance/security rules, the test + docs steps, any generated-artifact
  regeneration, final review/verification passes.

## 2. Round structure & dependencies
- Re-derive the dependency graph from each phase's "Dependencies" + "Files".
  Is the topological sort into rounds correct? Could any phase placed in a
  parallel round actually depend (by data or by file) on another phase in the
  same round?
- Check each stated cross-phase dependency: is it real and complete, or
  overstated/missing? A missing dependency causes a teammate to start before
  its inputs exist; a false one serialises work that could parallelise.

## 3. File-conflict matrix (verify independently, don't trust)
- Re-derive each phase's Create/Modify file set from its "Files" section. For
  every round with more than one phase, confirm no two phases touch the same
  file. Re-check the plan's own matrix against your derivation.
- Scrutinise any "apparent overlap, not a real conflict" claim the plan makes
  — confirm the reasoning holds (e.g., one phase edits a file at runtime in an
  ephemeral workspace vs. another editing the tracked file). Call out any real
  hazard the plan hand-waved.

## 4. Phase self-containment & executability
- For each phase, ask: could a fresh teammate with NO other context execute it
  from this file alone? Are file paths, line refs, code/config snippets, exact
  commands, and verification criteria all present and concrete — or are there
  placeholders or "see the master plan" hand-waves that force reconstruction?
- Are the atomic steps right-sized (one action each) and test-first where the
  phase delivers code? Is the project's test discipline (e.g., the test
  framework's required setup, a mandatory guard/idiom) correctly propagated
  into the phases that need it?

## 5. file:line accuracy (spot-check against the repo)
- The phases inherit many `file:line` hints. Open a sample of the cited files
  and confirm the references resolve to the claimed surface area. Flag stale
  or wrong ones — a teammate will trust them blindly.

## 6. Right-sizing
- Is any phase too COARSE (bundles several independent units that should split,
  hurting single-focus and reviewability)? Too TRIVIAL (should merge)? Weigh
  the tension between single-focus phases and avoiding parallel file conflicts,
  and give a verdict on the borderline ones.

## 7. Owners, skills, and gaps
- Are the per-phase owner agents / specialists appropriate to the files each
  touches? Are missing-skill or tooling gaps recorded rather than ignored?

## 8. Honesty of outstanding items
- Are items that genuinely can't be completed by an automated teammate (manual
  edits, restricted files, a real deploy/CI run, an external approval) flagged
  as OUTSTANDING/blocked — not silently checked off as done?

## 9. Orchestrator & consistency
- Does the orchestrator/runner doc correctly encode the rounds (parallel
  dispatch within a round, sequential advance between rounds) and the project's
  execution rules? Are there unsubstituted placeholders that should be concrete
  values, vs. legitimate per-dispatch template slots?
- Cross-check consistency across the tracker, the orchestrator, the phase
  files, and the hand-off doc: phase numbers, titles, filenames, owners, round
  assignments, and dependency lists must all agree.

## 10. Executability risks (predict failures)
- Predict where a teammate would realistically BLOCK or FAIL: a phase that
  can't verify in isolation, a missing tool, an unreadable file, an ordering
  assumption that breaks. Flag each with the phase and the reason.

<Scrutinise especially: …  (add 0–3 bullets naming the riskiest parts of THIS
decomposition — a multi-unit phase, a parallel round, a cross-phase
dependency. Omit if none.)>
```

## Block 2 — Output format

```
# Output format

Produce:
1. **Verdict** — Approve / Approve-with-changes / Needs-rework, with a 2–3
   sentence rationale. <For an "all" review: give ONE combined verdict plus a
   one-line sub-verdict for the master plan and for the decomposition.>
2. **Coverage matrix** — every master-plan unit of work → the phase that
   implements it, with orphans (dropped) and inventions (unplanned) called out
   explicitly. This is the heart of the review.
3. **Findings table** — one row per finding:
   `Severity (Blocker/Major/Minor/Nit) | Location (file:line) | Issue |
   Evidence (the file:line you checked) | Recommended fix`. Sort by severity.
4. **Answers to the high-judgment questions** — explicitly rule on each named
   question (the parallel-round conflict-free claim, any multi-unit phase, each
   cross-phase dependency), with evidence.
5. **What the decomposition got right** — so revisions don't regress it.
6. **Executability risks** — the predicted block/fail points, and any open
   questions for the author.

Separate verified facts ("I read PHASE-0X and <file:line>; the reference
resolves") from opinions ("I'd split this phase"). If you can't verify a
reference because a file is missing, say so rather than assuming. Where the
decomposition merely reflects a master-plan decision you'd question, note it as
out-of-scope for this review.
```
