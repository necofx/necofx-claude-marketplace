# Implementation review — instructions + output format (embeddable)

This file holds the two blocks the skill embeds into a generated Codex prompt that reviews the **implementation** of a plan: the git changeset judged against the planning folder, and the code judged on its own merits.

**How to use:** copy the two fenced blocks below into the assembled prompt's "How to perform the review" and "Output format" sections. Tailor the `<...>` placeholders — the real stack, the real build/test commands, the diff base ref. Where you spotted a high-risk change during onboarding (a security-sensitive edit, a shared-function change with wide blast radius, a keystone everything else depends on), add a "Scrutinise especially: …" bullet so the reviewer spends effort where it matters. Keep everything else verbatim — the checklist is deliberately general.

**Tailoring notes:**
- Substitute the project's real **build/test commands** so the reviewer can confirm the change builds and the tests actually exercise it.
- The two dimensions are deliberately separate: **Dimension 1** asks "does the code match the plan", **Dimension 2** asks "is the code good regardless of the plan". A change can pass one and fail the other — keep the axes distinct in the findings.
- Name explicitly any item the plan flagged as **deferred / manual / restricted**, and any phase the tracker marks **"completed"**, so the reviewer must rule on whether reality matches the claim.
- The reviewer judges the *implementation*, not the plan's strategy. Where the code merely reflects a settled plan decision, that is out-of-scope for this review (it belongs to a pre-build plan review).

---

## Block 1 — How to perform the review

```
# How to perform the review

You are judging an implementation on two independent axes: (1) does the
changeset faithfully and completely build what the plan specified, and (2) is
the code correct, safe, and up to the project's standards on its own merits.
A change can satisfy one axis and fail the other — keep them distinct.

First, reproduce the changeset live (do not trust any pasted diff):
`git status --porcelain`, `git diff <base>` (the form named above), and read
every untracked file in full. If your live `git diff <base> --stat` differs from
the snapshot in the onboarding, the working tree changed since this prompt was
written — review the live state and note the drift.

## Dimension 1 — Plan conformance (does the code match the plan?)

### 1a. Forward traceability — what's missing
- Build a map: every unit of work in the plan (each finding / requirement /
  acceptance criterion / phase atomic step) -> the change that implements it.
  List every plan item with NO corresponding change as UNIMPLEMENTED, and every
  partially-done item as PARTIAL with what's left.
- The onboarding gives you a navigational changed-file -> plan-item map. It tells
  you WHERE to look; it does NOT tell you whether the work is done. Decide
  done / partial / missing yourself, from the code.

### 1b. Backward traceability — what's unplanned
- For every change in the diff, find the plan item that authorizes it. Flag
  changes with NO plan basis: either scope creep to remove, or necessary work
  the plan failed to capture (say which). Unplanned edits to files the plan
  never mentions are the usual tell.

### 1c. Fidelity — did it build what the plan said, the way it said
- Where the plan specified the change EXACTLY (a precise snippet, a signature,
  a config key, a guard condition, a file:line edit), confirm the landed code
  matches it. A deviation is a finding to RULE ON, not merely note: either the
  code is wrong, or the plan was wrong and should be updated to match — say
  which, with evidence.

### 1d. Acceptance criteria
- Walk each acceptance criterion in the spec and judge whether the changeset
  actually SATISFIES it — not merely "a relevant file was touched". Cite the
  code that meets it, or flag the gap.

### 1e. Honesty of status & outstanding items
- The tracker (tasks.md or equivalent) claims phase statuses. For each phase
  marked "completed", confirm the diff actually backs that claim; a phase marked
  done with no corresponding change is a finding.
- Items the plan flagged as deferred / manual / restricted / developer-dependent
  should be ABSENT from the diff and recorded as outstanding — confirm they were
  neither silently faked nor silently dropped.

## Dimension 2 — Code quality & technical correctness (regardless of the plan)

Read the post-change files in full (onboarding group C) — a diff hunk hides the
surrounding code, the callers, and the error paths.

### 2a. Correctness
- Walk the critical path of the new/changed code. Look for logic errors,
  off-by-one, null/empty/boundary cases, wrong ordering, and unhandled error
  paths. Identify the keystone changes (the ones the rest depends on) and
  scrutinise them hardest.

### 2b. Does it actually work
- Would <build cmd> compile clean and <test cmd> pass? Confirm the change is
  internally complete — no dangling reference, no half-renamed symbol, no import
  or dependency that doesn't resolve.

### 2c. Regressions & blast radius
- For each modified shared function / type / config, check its callers and
  dependents. Does the change break an existing contract, caller, or behavior?
  Trace the blast radius; a local fix with a non-local break is the classic miss.

### 2d. Security
- New attack surface (a new endpoint, port, parameter, deserialization,
  file/path handling, query), injection, authz gaps, unsafe defaults, and
  secrets or credentials in code or config. Treat new inputs as hostile.

### 2e. Project standards (discovered, not assumed)
- Read the standards files in onboarding group D and hold the code to the
  project's ACTUAL conventions — naming, typing, formatting, async/error
  patterns, logging, and any output/log-hygiene rule the repo enforces. Cite the
  rule file and the violating line. Do not impose conventions the repo doesn't
  state.

### 2f. Tests
- Are there tests for the new behavior, and do they actually EXERCISE it (assert
  the new logic, not vacuously pass)? Did the change update or break existing
  tests? Are the project's test idioms (framework, required setup, mandatory
  guards) followed?

### 2g. Docs & generated artifacts
- Were the docs, READMEs, schemas, or generated files the change or the
  standards require updated — and are generated files regenerated by their tool,
  not hand-edited?

### 2h. Hygiene
- Leftover debug code, commented-out blocks, TODOs without tracking, stray
  print/log lines, and anything the repo's conventions forbid in committed code.

<Scrutinise especially: …  (add 0–3 bullets naming the highest-risk changes in
THIS diff — a security-sensitive edit, a wide-blast-radius change, a keystone.
Omit if none.)>
```

## Block 2 — Output format

```
# Output format

Produce:
1. **Verdict** — Ship / Fix-before-commit / Rework, with a 2–3 sentence
   rationale. (The changeset is pending; the bar is "safe and complete to
   commit".)
2. **Conformance matrix** — every plan unit of work -> its status
   (Implemented / Partial / Missing) with the file:line that implements it (or
   "no change found"). Then a short list of UNPLANNED changes (in the diff, not
   in the plan). This is the heart of Dimension 1; do not skip an item.
3. **Findings table** — one row per finding:
   `Severity (Blocker/Major/Minor/Nit) | Dimension (Conformance/Correctness/
   Security/Standards/Tests/Docs) | Location (file:line or diff hunk) | Issue |
   Evidence (the file:line you read) | Recommended fix`. Sort by severity.
4. **Status-honesty check** — for each phase the tracker marks "completed" and
   each item flagged deferred/manual/restricted: does reality match the claim?
   Name any mismatch.
5. **Deviation rulings** — for each place the code departs from the plan's exact
   spec, your ruling: code-is-wrong (fix the code) or plan-was-wrong (update the
   plan), with evidence.
6. **What's correct** — the parts that are right, so revisions don't regress
   them.
7. **Open questions** for the author, if any.

Separate verified facts ("the plan said X; <file:line> does X") from opinions
("I'd refactor Z"). If you cannot verify a claim because a file is missing or
the diff doesn't include it, say so rather than assuming. Where the code merely
reflects a settled plan decision you would have made differently, note it as
out-of-scope for this review — judge the implementation, not the plan's strategy.
```
