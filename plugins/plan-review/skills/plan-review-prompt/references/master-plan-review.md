# Master-plan review — instructions + output format (embeddable)

This file holds the two blocks the skill embeds into a generated Codex prompt when the review target is **master plan** (or as "Part 1" when the target is **all**).

**How to use:** copy the two fenced blocks below into the assembled prompt's "How to perform the review" and "Output format" sections. Tailor the `<...>` placeholders and, where you noticed plan-specific high-risk areas during onboarding discovery (a tricky algorithm, a security-sensitive change, a load-bearing assumption), add a short "Scrutinise especially: …" bullet so the reviewer spends effort where it matters. Keep everything else verbatim — the checklist is deliberately general.

**Tailoring notes:**
- Substitute the real **stack** and **build/test commands** (from the plan's Tech Stack section) into the technical-correctness and completeness items.
- The reviewer should judge the *plan*, not re-decide product strategy. The instructions already say this — keep it.
- If the plan has a "flagged for reviewer" decision (the author was unsure), name it explicitly in the high-judgment-questions list so the reviewer must answer it.

---

## Block 1 — How to perform the review

```
# How to perform the review

Review the master plan for correctness, completeness, faithfulness to the
actual repo, and executability. Judge the PLAN — not the underlying product
or architecture decisions, which were settled elsewhere. Where the plan
contradicts the source-of-truth spec, flag it; where you merely disagree with
a settled decision, note it as out-of-scope for this review.

## 1. Fidelity to ground truth (verify, don't trust)
- Open the files in onboarding group C and confirm that EVERY `file:line`
  reference in the plan resolves to the surface area the plan claims.
  List any that are wrong, stale, or off by more than a few lines.
- Confirm the plan's statements about the current code/system are TRUE
  (e.g., "function X does Y today", "the script only handles Z"). Read the
  code and verify. Flag any claim the code contradicts.
- Watch for stale assumptions — references to a tool, branch, file, or
  approach that the repo has since changed or retired.

## 2. Technical correctness
- Does the proposed approach actually achieve the stated goal? Walk the
  critical path end-to-end and confirm each step does what the plan assumes.
- Identify the load-bearing "keystone" steps — the ones everything else
  depends on — and scrutinise them hardest. If a keystone is wrong, say so
  with evidence.
- Check edge cases, error paths, ordering/sequencing assumptions, and any
  step that could break the build, the tests, or a runtime contract.
- Substitute the project's real build/test commands (<build cmd> / <test cmd>)
  and confirm the plan's verification steps would actually exercise the change.

## 3. Soundness of the key decisions
- The plan makes high-judgment calls (scope, architecture, sequencing,
  trade-offs). For each major one, decide: is it justified by evidence in the
  repo / spec, or is it asserted? Flag under-justified or wrong calls.
- Pay special attention to anything the plan itself flags as "confirm with
  reviewer" or "superseded" — give a definite verdict with evidence.

## 4. Completeness / coverage
- Map every requirement / acceptance criterion in the source-of-truth spec to
  plan work. List anything DROPPED (a requirement with no corresponding plan
  step) and anything INVENTED (plan work not traceable to a requirement).
- Confirm the easily-forgotten cross-cutting concerns are covered: tests,
  docs, security/compliance, logging/observability, rollback/back-compat,
  and any "definition of done" the project enforces.

## 5. Internal consistency & executability
- Are names, signatures, paths, and terminology consistent across all
  sections (a type/function/flag named one way in one section and another way
  elsewhere is a bug)?
- Is the plan concrete enough to execute? Flag placeholders, "TBD", "add
  appropriate handling", or any step that says WHAT without HOW where HOW is
  needed.
- Are risks and out-of-scope explicit and believable?

<Scrutinise especially: …  (add 0–3 bullets naming the plan-specific
high-risk areas you noticed during onboarding — omit if none.)>
```

## Block 2 — Output format

```
# Output format

Produce:
1. **Verdict** — Approve / Approve-with-changes / Needs-rework, with a 2–3
   sentence rationale.
2. **Coverage check** — a short list mapping each spec requirement /
   acceptance criterion to the plan step that covers it, calling out any
   orphan (dropped) or invented item explicitly.
3. **Findings table** — one row per finding:
   `Severity (Blocker/Major/Minor/Nit) | Location (file:line) | Issue |
   Evidence (the file:line you checked) | Recommended fix`. Sort by severity.
4. **Answers to the high-judgment questions** — explicitly resolve each
   decision the plan flagged for reviewer confirmation (and any other call you
   judge load-bearing), with evidence.
5. **What the plan got right** — so revisions don't regress it.
6. **Open questions** for the author, if any.

Separate verified facts ("I read X:NN, it says Y") from opinions ("I'd prefer
Z"). If you cannot verify a claim because a file is missing or unreadable, say
so rather than assuming. Where the plan reflects a settled product decision you
would have made differently, note it as out-of-scope rather than a defect.
```
