# close-master-plan Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the missing sixth step to the master-plan pack — a `close-master-plan` plugin that reconciles, verifies, distils and archives a finished plan — and adapt `create-master-plan` and the pack's documentation to the `active/` + `closed/` layout it introduces.

**Architecture:** Everything here is markdown and JSON; there is no executable product code. A "plugin" is a `plugin.json` manifest plus `skills/<name>/SKILL.md` instructions and `references/*.md` documents that an agent reads at runtime. Correctness therefore means the instructions are unambiguous, internally consistent, and consistent with the files they claim to modify. Verification is a throwaway git fixture the skill is run against, plus a mechanical consistency checker over the repo's own prose.

**Tech Stack:** Markdown · JSON manifests · bash (fixture + checker) · `mermaid-cli` driven by the system Chrome · `claude plugin` CLI for the install smoke test.

**Spec:** `docs/superpowers/specs/2026-08-18-close-master-plan-design.md`

## Global Constraints

Copied verbatim from the spec. Every task's requirements implicitly include this section.

- **Plans root:** `<plans-root>` defaults to `docs/plans/`. A `CLAUDE.md` declaration naming a directory is the root; one that still ends in `<TICKET-ID>` has that final segment stripped and the remainder is the root.
- **Status vocabulary, exact strings:** `completed` · `abandoned` · `superseded by <TICKET-ID>`. There is no `merged` status.
- **Status header, exact format**, in the first five lines of `master-plan.md` (and of `tasks.md` / `handoff.md` when they exist):
  `<!-- STATUS: completed · closed 2026-08-18 · PR #77 · rules: .claude/rules/java.md -->`
  Separator is ` · ` (space, U+00B7, space). Absent values are `PR —` and `rules: —`.
- **Authoritative carrier:** only `master-plan.md` is read for status. `issue.specs` and `phases/*.md` are never stamped.
- **Collapsed scan line, exact format:**
  `- GH-388 · superseded by GH-412 · closed 2026-06-02 — matched: idempotency, refund`
- **Dates are absolute** `YYYY-MM-DD`, never relative.
- **Staging:** always explicit paths. Never `git add -A` — `tasks-template.md`'s hard rules forbid it.
- **No network calls** anywhere in the new skill. No `gh`, no PR comment, no issue close.
- **No git state changes** by the skill: it does not commit, push, switch branches, or remove worktrees. It writes files and prints commands.
- **`MANUAL.html` stays byte-identical across all four plugins.** Edit once, copy to four, verify with `md5sum`.
- **Manifest author block** for the new plugin, matching the other three: `{"name": "necofx", "email": "admin@necofx.com"}`.
- **Mermaid:** never a `;` or `<...>` inside a label or note — a `;` is a statement separator and truncates the diagram. Every new block is render-checked before the commit that adds it.
- **Exercising the skill under construction:** install it from the local path once — `claude plugin marketplace add ./` then `/plugin install close-master-plan@necofx` — and re-add the marketplace if an edit does not show up. If installation is in the way, the fallback is exact and always works: read the in-progress `SKILL.md` and follow it literally against the fixture. The verification steps below assert outcomes in the fixture, so either route satisfies them.
- **Commits:** each task ends with a proposed commit. The executor runs it only with the user's go-ahead; nothing in this plan authorises committing unprompted.

---

## File Structure

**Created — the new plugin**

| File | Responsibility |
|---|---|
| `plugins/close-master-plan/.claude-plugin/plugin.json` | Manifest, v0.1.0 |
| `plugins/close-master-plan/README.md` | User-facing docs: what it does, when to run it, the two-part `abandoned` ending, why there is no reopen |
| `plugins/close-master-plan/MANUAL.html` | Copy of the shared manual (Task 9) |
| `plugins/close-master-plan/skills/close-master-plan/SKILL.md` | The nine-step workflow |
| `…/references/plan-layout.md` | §3 of the spec. Shipped byte-identical in `create-master-plan` too |
| `…/references/closeout-checklist.md` | What must be complete in `tasks.md` / `handoff.md`; the placeholder token set |
| `…/references/rule-distillation.md` | How to read Deviations/Decisions and propose rules; the durability test; worked examples |
| `…/references/index-template.md` | `INDEX.md` shape |

**Modified**

| File | Change |
|---|---|
| `plugins/create-master-plan/skills/create-master-plan/SKILL.md` | Step 1 path (line 42), Step 5 partitioned reporting (91-105), Step 6 section renumbering (125-133) |
| `plugins/create-master-plan/skills/create-master-plan/references/issue-specs-template.md` | `### Closed plans` subsection under *Related local docs* |
| `plugins/create-master-plan/skills/create-master-plan/references/plan-layout.md` | New — byte-identical copy |
| `plugins/create-master-plan/README.md` | Flat layout at lines 41, 78, 111 |
| `plugins/create-master-plan/config.md` | The knob becomes a root |
| `plugins/create-master-plan/.claude-plugin/plugin.json` | 0.3.4 → 0.4.0 |
| `plugins/decompose-plan/**/*.md`, `plugins/plan-review/**/*.md` | Path examples; patch bumps |
| `plugins/*/MANUAL.html` ×4 | Five steps → six; layout refs |
| `README.md` | Fourth row, workflow diagram, "The three are one workflow" (line 60), version numbers |
| `.claude-plugin/marketplace.json` | Fourth entry |
| `docs/master-plan-pack/TUTORIAL.md` | 22 path refs, line 3, line 86, new Part 6, new diagram, cheat sheet |

**Verification artifacts — scratchpad only, never committed**

`$SCRATCH/fixture.sh` (builds the throwaway repo) and `$SCRATCH/check.sh` (repo consistency assertions), where `$SCRATCH` is this session's scratchpad directory.

---

### Task 1: The verification harness

Build the checks before the thing they check, so every later task has a gate that is already known to fail for the right reason.

**Files:**
- Create: `$SCRATCH/fixture.sh`
- Create: `$SCRATCH/check.sh`

**Interfaces:**
- Consumes: nothing.
- Produces: `fixture.sh <dir>` builds a throwaway git repo at `<dir>` on branch `feature/gh-999` with a complete `docs/plans/active/GH-999/` and three round commits. `check.sh <repo-root>` exits 0 when every repo-consistency assertion holds, non-zero otherwise, printing one line per failed assertion.

- [ ] **Step 1: Write the fixture builder**

```bash
cat > "$SCRATCH/fixture.sh" <<'EOF'
#!/usr/bin/env bash
# fixture.sh <dir> — throwaway repo for exercising /close-master-plan
set -euo pipefail
DIR="${1:?usage: fixture.sh <dir>}"
rm -rf "$DIR"; mkdir -p "$DIR/docs/plans/active/GH-999/phases"
cd "$DIR"
git init -q -b main
git config user.email fixture@example.com
git config user.name Fixture
mkdir -p src .claude/rules
echo '# java rules' > .claude/rules/java.md
echo 'placeholder' > src/app.java
git add -A && git commit -qm "initial"
git switch -qc feature/gh-999

P=docs/plans/active/GH-999
cat > $P/master-plan.md <<'MP'
# GH-999 — Fixture plan

## Why
Exercise the close-out skill.
MP
cat > $P/issue.specs <<'IS'
# GH-999 · Fixture · open · github · 2026-08-18 10:00

## 7. Related local docs
- docs/architecture/thing.md — matched: fixture
IS
cat > $P/tasks.md <<'TK'
# GH-999 — Task Tracker

**Last updated:** 2026-08-18 — by coordinator (plan decomposition)

| # | Phase | Round | Owner agent | Status | Agent | Started | Finished | Commits | PR |
|---|---|---|---|---|---|---|---|---|---|
| 00 | Foundation | 0 | java-pro | completed | phase-00 | 2026-08-18 10:00 | 2026-08-18 10:40 | (pending batch) | — |
| 01 | Service | 1 | java-pro | completed | phase-01 | 2026-08-18 10:45 | 2026-08-18 11:30 | (pending batch) | — |
| 02 | Wiring | 2 | java-pro | completed | phase-02 | 2026-08-18 11:35 | 2026-08-18 12:10 | (pending batch) | — |

## Detailed Progress

### Phase 00 — Foundation
- 2026-08-18 10:40 — done

### Phase 01 — Service
- 2026-08-18 11:30 — done

### Phase 02 — Wiring
- 2026-08-18 12:10 — done

## Decisions

_none yet_

## Coordination Notes

- Round 1 used the existing amount parameter rather than adding a second.

## Final Summary

_(Written by the coordinator once every phase reaches `completed`.)_
TK
cat > $P/handoff.md <<'HO'
# GH-999 — Hand-off for code review

**Last touched:** 2026-08-18
**Branch:** `{branch-name}`
**Status:** _(to be filled by the coordinator after the last round)_

## What this iteration delivered

1. _(to be filled)_

## Background docs (read in this order before reviewing code)

1. `{MASTER_PLAN_FILENAME}` — architecture, phase index.

## Key deviations from the original plan (worth scrutinising)

1. _(to be filled)_

Total: {N} new tests.
HO
echo '# Phase 00' > $P/phases/PHASE-00-foundation.md
echo '# Phase 01 — fixture idempotency notes' > $P/phases/PHASE-01-service.md
git add docs/plans && git commit -qm "GH-999: plan"

for n in 00 01 02; do
  echo "round $n" >> src/app.java
  git add src/app.java && git commit -qm "GH-999 round $n: phase $n"
done
echo "fixture ready at $DIR (branch feature/gh-999, base main)"
EOF
chmod +x "$SCRATCH/fixture.sh"
```

- [ ] **Step 2: Write the consistency checker**

```bash
cat > "$SCRATCH/check.sh" <<'EOF'
#!/usr/bin/env bash
# check.sh <repo-root> — repo-wide consistency assertions
set -uo pipefail
R="${1:?usage: check.sh <repo-root>}"; cd "$R"; fail=0
say(){ echo "FAIL: $*"; fail=1; }

# 1 — marketplace lists four plugins and every source exists
n=$(grep -c '"source": "./plugins/' .claude-plugin/marketplace.json)
[ "$n" -eq 4 ] || say "marketplace.json lists $n plugins, expected 4"
while read -r d; do
  [ -d "$d" ] || say "marketplace source $d does not exist"
done < <(grep -o './plugins/[a-z-]*' .claude-plugin/marketplace.json | sort -u)

# 2 — README version table matches the manifests
for f in plugins/*/.claude-plugin/plugin.json; do
  p=$(basename "$(dirname "$(dirname "$f")")")
  v=$(grep -o '"version": "[^"]*"' "$f" | cut -d'"' -f4)
  grep -q "\`$p\`.*| $v |" README.md || say "README version for $p is not $v"
done

# 3 — MANUAL.html byte-identical across all four
c=$(md5sum plugins/*/MANUAL.html | awk '{print $1}' | sort -u | wc -l)
[ "$c" -eq 1 ] || say "MANUAL.html differs across plugins ($c distinct hashes)"

# 4 — plan-layout.md byte-identical in both plugins that carry it
a=plugins/close-master-plan/skills/close-master-plan/references/plan-layout.md
b=plugins/create-master-plan/skills/create-master-plan/references/plan-layout.md
cmp -s "$a" "$b" || say "plan-layout.md differs between the two plugins"

# 5 — every flat docs/plans/<ID> reference is explicitly about the legacy layout
while read -r l; do say "orphan flat-layout reference: $l"; done < <(
  grep -rn 'docs/plans/GH-' --include='*.md' --include='*.html' . 2>/dev/null \
    | grep -v '/docs/superpowers/' \
    | grep -viE 'legacy|flat layout'
)

# 6 — no surviving five-step or three-plugin wording
while read -r l; do say "stale count wording: $l"; done < <(
  grep -rniE 'the five steps|all five steps|three plugins|the three are one' \
    README.md docs/master-plan-pack/TUTORIAL.md plugins/*/README.md plugins/*/MANUAL.html 2>/dev/null
)

# 7 — the new plugin's required files exist
for f in plugins/close-master-plan/.claude-plugin/plugin.json \
         plugins/close-master-plan/README.md \
         plugins/close-master-plan/skills/close-master-plan/SKILL.md \
         plugins/close-master-plan/skills/close-master-plan/references/plan-layout.md \
         plugins/close-master-plan/skills/close-master-plan/references/closeout-checklist.md \
         plugins/close-master-plan/skills/close-master-plan/references/rule-distillation.md \
         plugins/close-master-plan/skills/close-master-plan/references/index-template.md; do
  [ -f "$f" ] || say "missing $f"
done

[ $fail -eq 0 ] && echo "check.sh: all assertions passed"
exit $fail
EOF
chmod +x "$SCRATCH/check.sh"
```

- [ ] **Step 3: Run both to verify they fail for the right reasons**

Run: `"$SCRATCH/fixture.sh" "$SCRATCH/fx" && "$SCRATCH/check.sh" "$PWD"`
Expected: the fixture reports `fixture ready at …`; the checker prints `FAIL: marketplace.json lists 3 plugins, expected 4`, `FAIL: missing plugins/close-master-plan/…` for each of the seven files, plus the stale-count and version-table failures. It must NOT print `all assertions passed`.

- [ ] **Step 4: Sanity-check the checker itself**

Run: `sed -i 's/expected 4/expected 99/' "$SCRATCH/check.sh"` then re-run, confirm the message changes, then revert with `sed -i 's/expected 99/expected 4/'`.
Expected: a checker that never fails, or fails identically no matter what, proves nothing. Confirm assertion 1 responds to its own threshold.

- [ ] **Step 5: No commit**

The harness lives in the scratchpad by design (spec §8) and is never committed.

---

### Task 2: Plugin skeleton and the shared layout reference

**Files:**
- Create: `plugins/close-master-plan/.claude-plugin/plugin.json`
- Create: `plugins/close-master-plan/skills/close-master-plan/references/plan-layout.md`
- Create: `plugins/create-master-plan/skills/create-master-plan/references/plan-layout.md` (identical copy)

**Interfaces:**
- Consumes: Task 1's `check.sh` assertions 4 and 7.
- Produces: `plan-layout.md`, the single normative source both plugins cite for the root-resolution rule, the status vocabulary, the header format and its authoritative carrier, and the index format. Later tasks reference it by name rather than restating it.

- [ ] **Step 1: Write the manifest**

```json
{
  "name": "close-master-plan",
  "version": "0.1.0",
  "description": "Step 6 of a plan-first multi-agent workflow: reconcile tasks.md with the real commits, verify handoff.md is complete, distil the run's durable lessons into .claude/rules/, stamp the plan's outcome and archive it under docs/plans/closed/, then print the commit and stop.",
  "author": {
    "name": "necofx",
    "email": "admin@necofx.com"
  }
}
```

- [ ] **Step 2: Write `plan-layout.md`**

Its content is §3 of the spec (`docs/superpowers/specs/2026-08-18-close-master-plan-design.md`, sections 3.1, 3.2, 3.3), transposed from design prose into reference prose — present tense, addressed to the agent reading it, no "the design says". It must state, in this order:

1. The directory tree, with `<plans-root>` named as a variable and `docs/plans/` as its default.
2. The root-resolution rule, both branches (directory declaration → root; declaration ending in `<TICKET-ID>` → strip that segment).
3. The status vocabulary, all three values verbatim, and the sentence explaining why there is no `merged`.
4. The header format with the field table, the ` · ` separator, and the `PR —` / `rules: —` absent forms.
5. **The authoritative-carrier rule**: only `master-plan.md` is stamped and read; a match on any file under `closed/<ID>/` resolves status from `closed/<ID>/master-plan.md`; one lookup per folder; missing or headerless → status `unknown`, never skipped.
6. The `INDEX.md` table shape, newest first, and the note that nothing in either plugin's behaviour depends on it.

- [ ] **Step 3: Copy it to the second plugin byte-identically**

Run: `cp plugins/close-master-plan/skills/close-master-plan/references/plan-layout.md plugins/create-master-plan/skills/create-master-plan/references/plan-layout.md`

- [ ] **Step 4: Verify**

Run: `"$SCRATCH/check.sh" "$PWD" 2>&1 | grep -E 'plan-layout|missing plugins/close-master-plan/\.claude-plugin'`
Expected: no `plan-layout.md differs` line, and no `missing …/plugin.json` line. The other `missing …` lines still appear — later tasks create those files.

- [ ] **Step 5: Commit**

```bash
git add plugins/close-master-plan/.claude-plugin/plugin.json \
        plugins/close-master-plan/skills/close-master-plan/references/plan-layout.md \
        plugins/create-master-plan/skills/create-master-plan/references/plan-layout.md
git commit -m "close-master-plan: plugin skeleton and shared plan-layout reference"
```

---

### Task 3: SKILL.md steps 1–3 — locate, git preflight, status

**Files:**
- Create: `plugins/close-master-plan/skills/close-master-plan/SKILL.md`

**Interfaces:**
- Consumes: `references/plan-layout.md` from Task 2.
- Produces: the skill's frontmatter (`name: close-master-plan`, a `description` following the house pattern — third person, states when to trigger, lists the trigger phrases) and Steps 1–3. Tasks 4–6 append Steps 4–9 to this same file.

- [ ] **Step 1: Write the frontmatter and Overview**

Match the shape of `plugins/create-master-plan/skills/create-master-plan/SKILL.md:1-20`: `---` frontmatter with `name` and a `description` that ends with the trigger sentence (`Trigger when the user invokes /close-master-plan …, or asks to "close the plan for GH-412", "archive this plan", "wrap up GH-412"`), then `# Close Master Plan`, `## Overview`, `## Inputs`, `## Workflow`.

The Overview must state the two things a reader needs before anything else: this runs **after** `/code-review` and `/plan-implementation-review`, and it never changes git state.

- [ ] **Step 2: Write Step 1 — Locate and validate**

Required content: resolve project root by nearest ancestor `.git`; resolve `<plans-root>` per `references/plan-layout.md`; look for `<plans-root>/active/<ID>/` then fall back to flat `<plans-root>/<ID>/`; **record which layout was found, because Step 8 needs it**; require `master-plan.md` or stop. No-argument behaviour: exactly one folder in `active/` → propose and confirm; several → ask; empty → try flat; nothing → stop.

- [ ] **Step 3: Write Step 2 — Git preflight**

Required content, in this order: is this a git repo (stop if not); is the working tree clean (**stop if not**, with the reason — the close commit must contain only the close); worktree detection via `git rev-parse --git-dir` ≠ `--git-common-dir`; base branch via `git merge-base`, ask if ambiguous, citing that on a committed branch the base is not `HEAD`; `git log --oneline --no-merges <base>..HEAD` as Step 4's raw material.

Then the two non-blocking warnings: the main-checkout/worktree mismatch, and the absent implementation review (stating that review is optional, so this informs rather than gates).

- [ ] **Step 4: Write Step 3 — Establish the status**

Required content: `AskUserQuestion` with the three values verbatim; `superseded` prompts for the superseding id. Then the `abandoned` paragraph: the branch is about to be deleted unmerged, so a close commit made on it is discarded with it and `closed/<ID>/` never reaches the default branch — losing the record this step exists to preserve. The close still happens here because the folder's content only lives on this branch, but Step 9 prints a two-part sequence and says the close is not durable until the second part runs.

- [ ] **Step 5: Verify against the fixture**

Run the skill against `$SCRATCH/fx` and assert, one at a time:
1. With no argument it proposes `docs/plans/active/GH-999` and asks for confirmation.
2. `touch $SCRATCH/fx/src/dirty.java` then re-run → it **stops** at Step 2 naming the dirty tree. Remove the file after.
3. It reports base `main`, branch `feature/gh-999`, and lists exactly the three round commits plus the plan commit.
4. It warns that no implementation review is present, and continues rather than stopping.
5. It offers exactly the three status values and no `merged`.

- [ ] **Step 6: Commit**

```bash
git add plugins/close-master-plan/skills/close-master-plan/SKILL.md
git commit -m "close-master-plan: locate, git preflight and status steps"
```

---

### Task 4: SKILL.md steps 4–5 — reconcile and verify

**Files:**
- Modify: `plugins/close-master-plan/skills/close-master-plan/SKILL.md` (append Steps 4, 4a, 5)
- Create: `plugins/close-master-plan/skills/close-master-plan/references/closeout-checklist.md`

**Interfaces:**
- Consumes: Step 2's commit list.
- Produces: a reconciled `tasks.md` and a verified `handoff.md`; `closeout-checklist.md`, which Step 5 cites for the placeholder token set.

- [ ] **Step 1: Write `closeout-checklist.md`**

Two sections. **`tasks.md` is complete when**: no `(pending batch)` remains except rows the user declined to place; every `completed` phase has a `Finished` timestamp; no phase is `pending`/`in_progress`/`blocked`; `Final Summary` is written; `**Last updated:**` is today.

**`handoff.md` is complete when** none of these survive — and list them as literal tokens so the scan is mechanical, not interpretive:

```
{PLAN_NAME}  {PLAN_SLUG}  {PLAN_KEY}  {MASTER_PLAN_FILENAME}  {branch-name}
{base-branch}  {N}  {NN}  {Phase NN Title}  {Component 1}  {owner-agent}
_(to be filled)_        _(none yet)_        _(updates appended by …)_
_(to be filled by the coordinator after the last round)_
```

State the general rule after the list, because the templates will grow: **any surviving `{...}` token and any italic parenthetical `_(...)_` is a hit.**

- [ ] **Step 2: Write Step 4 — Reconcile `tasks.md`**

The mapping rule is the part that must not be hand-waved. Required content:

> Build the best phase-to-commit mapping you can from the Detailed Progress entries and the commit subjects, **present it as a table, and have the user correct it.** Do not derive it silently: `tasks-template.md` requires the coordinator to replace `(pending batch)` with SHAs and to note which phases a batch covered, but it imposes **no commit-message format**, and the tutorial presents one-commit-per-round as what the coordinator *proposes*, not a contract. Rows the user cannot place stay `(pending batch)` rather than being guessed.

Then: fill `Finished` where missing on `completed` rows; fill `PR` if supplied; write `Final Summary` ending with the standing caveat — **the SHAs are feature-branch commits; if this PR is squash-merged they will not exist on the default branch, and the PR number is the durable pointer**; update `**Last updated:**`.

Then the stop-and-ask: any phase still `pending`/`in_progress`/`blocked` → mark `dropped` with a justification line under `Decisions` (the template's own mechanism), or abort.

- [ ] **Step 3: Write Step 4a — No `tasks.md`**

Required content: the plan was never decomposed; skip Steps 4 and 5; restrict the status to `abandoned` or `superseded`, because a plan never phased out cannot be `completed`; Step 7 stamps only the files that exist.

- [ ] **Step 4: Write Step 5 — Verify `handoff.md`**

Required content: scan for everything in `closeout-checklist.md`; report every hit and offer to fill them together before continuing. Then the deviations rule: an empty *Key deviations* section is not an automatic failure — a plan can be executed exactly as written — but it is confirmed out loud, because it is the section reviewers spend the most attention on.

- [ ] **Step 5: Verify against the fixture**

1. The skill proposes a three-row mapping table for phases 00/01/02 against the three `GH-999 round NN` commits and **asks for confirmation** rather than writing SHAs directly.
2. Decline row 02 → `tasks.md` keeps `(pending batch)` on that row only.
3. `Final Summary` is written and contains the squash caveat.
4. The `handoff.md` scan reports **at least** `{branch-name}`, `{MASTER_PLAN_FILENAME}`, `{N}`, and three `_(to be filled)_` hits. A scan that reports only the `_(to be filled)_` tokens is the first-draft bug and fails this step.
5. Edit the fixture to set phase 02 `Status = in_progress`, re-run → it stops and offers `dropped` or abort.

- [ ] **Step 6: Commit**

```bash
git add plugins/close-master-plan/skills/close-master-plan/SKILL.md \
        plugins/close-master-plan/skills/close-master-plan/references/closeout-checklist.md
git commit -m "close-master-plan: reconcile tasks.md and verify handoff.md"
```

---

### Task 5: SKILL.md step 6 — distil durable lessons

**Files:**
- Modify: `plugins/close-master-plan/skills/close-master-plan/SKILL.md` (append Step 6)
- Create: `plugins/close-master-plan/skills/close-master-plan/references/rule-distillation.md`

**Interfaces:**
- Consumes: `handoff.md`'s deviations and `tasks.md`'s `Decisions` and `Coordination Notes`.
- Produces: zero or more approved edits to `.claude/rules/*.md`, and the list of files written, which Step 7 puts in the header's `rules:` field.

- [ ] **Step 1: Write `rule-distillation.md`**

Required content:

- **Where to read from**: `handoff.md` → *Key deviations from the original plan*; `tasks.md` → `Decisions` and `Coordination Notes`.
- **The durability test**, stated as the single gate: *would this still be true on the next ticket, in a different area of the codebase?* A one-off workaround fails it. A team convention passes it.
- **Which file to write to**: the rules file matching the stack, using the same detection as `create-master-plan`'s `tech-stack-profiles.md` (`.claude/rules/java.md`, `.claude/rules/dotnet.md`, …). If the repo has no `.claude/rules/`, offer to create the file; if declined, skip the step.
- **Two worked examples, one of each verdict.** Passes: *"Phase 03 named the Helm value `partialRefunds.enabled` because Helm values are camelCase by chart convention"* → a naming rule that holds for every future chart change. Fails: *"Phase 01 reused an unused amount parameter GH-388 had added"* → true once, about one method, teaches nothing forward.
- **The output shape**: each candidate is presented as a concrete diff against the target file, approved individually, and **nothing is written without an explicit yes**. Zero rules is a valid and common outcome — a run that taught nothing durable should teach nothing durable.

- [ ] **Step 2: Write Step 6 in SKILL.md**

Short, delegating to the reference: read the three sources, apply `references/rule-distillation.md`, present each candidate as a diff, approve one at a time, record the files written for Step 7.

- [ ] **Step 3: Verify against the fixture**

1. The fixture's `Coordination Notes` line ("used the existing amount parameter rather than adding a second") must be **rejected** by the durability test, and the skill must say why rather than silently dropping it.
2. Add to the fixture's `handoff.md` a deviation reading *"the Helm value is camelCase by chart convention, not the Java property name"*, re-run → it is proposed as a candidate against `.claude/rules/java.md` as a diff.
3. Decline it → `.claude/rules/java.md` is unchanged on disk (`git diff --exit-code .claude/rules`).
4. Accept it → the file gains exactly the approved text, and Step 7 later reports `rules: .claude/rules/java.md`.
5. A fixture with neither source populated produces zero candidates and says so, rather than inventing one.

- [ ] **Step 4: Commit**

```bash
git add plugins/close-master-plan/skills/close-master-plan/SKILL.md \
        plugins/close-master-plan/skills/close-master-plan/references/rule-distillation.md
git commit -m "close-master-plan: distil durable lessons into .claude/rules"
```

---

### Task 6: SKILL.md steps 7–9 — stamp, move, report

**Files:**
- Modify: `plugins/close-master-plan/skills/close-master-plan/SKILL.md` (append Steps 7, 8, 9 and §4.3's "Correcting a mistaken close")
- Create: `plugins/close-master-plan/skills/close-master-plan/references/index-template.md`

**Interfaces:**
- Consumes: the status from Step 3, the rules list from Step 6, the layout recorded by Step 1.
- Produces: the stamped and moved plan folder, an updated `INDEX.md`, and the printed commands. This is the last step; nothing consumes its output but the user.

- [ ] **Step 1: Write `index-template.md`**

The `INDEX.md` shape from `plan-layout.md` §3.3: `# Closed plans`, the six-column table (Plan · Title · Status · Closed · PR · Rules distilled), newest first, created on first close if absent.

- [ ] **Step 2: Write Step 7 — Stamp**

Required content: write the header (format in `plan-layout.md`) into `master-plan.md` — mandatory, Step 1 already required the file — and into `tasks.md` and `handoff.md` **if they exist**. `issue.specs` and `phases/*.md` are never stamped. If a header is already present, **replace it rather than adding a second**.

- [ ] **Step 3: Write Step 8 — Move and index**

The source is whichever layout Step 1 recorded; the destination is always `<plans-root>/closed/<ID>/`:

```sh
git mv docs/plans/active/GH-412 docs/plans/closed/GH-412   # active layout
git mv docs/plans/GH-412        docs/plans/closed/GH-412   # flat legacy layout
```

Required content beyond the commands: `git mv` because it stages the move and keeps the index coherent — it does **not** guarantee a rename-rendered diff, since rename display is Git's similarity detection and this close-out edits several of the files in the same commit that moves them; a clean rename block is likely, not promised. Untracked files (a plan never committed) → fall back to plain `mv` and say so. Destination already exists → stop and list, following `create-master-plan` Step 1's overwrite-aware pattern; never silently clobber. Then create or update `<plans-root>/INDEX.md`.

- [ ] **Step 4: Write Step 9 — Report and stop**

For `completed` and `superseded`:

```sh
git add docs/plans .claude/rules && git commit -m "GH-412: close plan"
```

For `abandoned`, the same commit **plus** the rescue, presented as required rather than optional:

```sh
git add docs/plans .claude/rules && git commit -m "GH-412: close plan (abandoned)"
# the branch is being deleted — the archive must also land on the default branch:
git switch main && git cherry-pick <that-commit>
```

Then: explicit paths, never `git add -A`; point at `superpowers:finishing-a-development-branch` for merging, pushing and worktree removal; and the closing statement that the skill does not commit, push, switch branches, or touch the worktree it is running inside.

- [ ] **Step 5: Write "Correcting a mistaken close"**

A short section, not a command: there is no reopen in v0.1.0. Correcting a close is `git revert` of that commit, or a manual reverse `git mv` plus removing the header and the index row. Say why: a reopen would have to reverse approved rules edits, restore a replaced header, and undo the `tasks.md` reconciliation — reversibility the skill cannot honestly claim.

- [ ] **Step 6: Verify against the fixture — happy path**

Run end to end on a clean `$SCRATCH/fx` and assert: `master-plan.md`, `tasks.md`, `handoff.md` each carry exactly one header, in the first five lines, in the exact format; `issue.specs` and `phases/*.md` carry none; the folder is at `docs/plans/closed/GH-999/`; `git status --short` shows staged renames; `INDEX.md` exists with one row; and the printed command is the single-commit form.

- [ ] **Step 7: Verify against the fixture — the four negative cases**

1. **Pre-existing destination**: `mkdir -p docs/plans/closed/GH-999` before running → stops and lists, moves nothing.
2. **Flat legacy layout**: rebuild the fixture with the folder at `docs/plans/GH-999/` → moves to `docs/plans/closed/GH-999/`, never requiring `active/` to exist.
3. **No `tasks.md`**: delete it → Steps 4–5 skipped, only `abandoned`/`superseded` offered, only `master-plan.md` and `handoff.md` stamped.
4. **`abandoned` on a feature branch**: → the printed output contains both the commit **and** the `git cherry-pick` line, plus the sentence that the close is not durable without it.

- [ ] **Step 8: Verify idempotence**

Run the skill a second time on the closed fixture → it reports when the plan was closed and stops; it does not move, re-stamp, or duplicate the `INDEX.md` row.

- [ ] **Step 9: Commit**

```bash
git add plugins/close-master-plan/skills/close-master-plan/SKILL.md \
        plugins/close-master-plan/skills/close-master-plan/references/index-template.md
git commit -m "close-master-plan: stamp, archive and report steps"
```

---

### Task 7: `create-master-plan` v0.4.0

**Files:**
- Modify: `plugins/create-master-plan/skills/create-master-plan/SKILL.md:42` (Step 1), `:91-105` (Step 5), `:125-133` (Step 6 numbering)
- Modify: `plugins/create-master-plan/skills/create-master-plan/references/issue-specs-template.md:80-94`
- Modify: `plugins/create-master-plan/README.md:41,78,111`
- Modify: `plugins/create-master-plan/config.md:52-62`
- Modify: `plugins/create-master-plan/.claude-plugin/plugin.json`

**Interfaces:**
- Consumes: `references/plan-layout.md`, already placed by Task 2.
- Produces: the other half of the cross-plugin contract — a Step 5 that reads `closed/<ID>/master-plan.md` for status and emits one collapsed line per closed plan folder.

- [ ] **Step 1: Step 1 — the write path**

Change `<plan-folder>` to `<plans-root>/active/<TICKET-ID>/`, resolved per `references/plan-layout.md`. Add: if `<plans-root>/<TICKET-ID>/` exists in the flat legacy layout, use it unchanged and note in the run summary that the project is on the old layout. The skill never migrates on its own.

- [ ] **Step 2: Step 5 — partitioned reporting**

The glob stays exactly as it is. Partition the *reporting*:

- Matches outside `<plans-root>`, and matches under `active/` — reported as today: relative path, matched terms, 1–2 line excerpt.
- Matches under `closed/` — **collapsed to one line per plan folder**, no excerpt, however many files inside it matched:
  `- GH-388 · superseded by GH-412 · closed 2026-06-02 — matched: idempotency, refund`
  Status read from that folder's `master-plan.md` header; missing or headerless → `unknown`, never skipped.

Add the sentence that keeps the intent honest: this reduces what the dossier carries, not what the scan reads — the glob still traverses every markdown file, and the cost being cut is context.

- [ ] **Step 3: `issue-specs-template.md` — the new subsection**

Under *Related local docs*, add `### Closed plans` holding the collapsed entries, kept separate from the full entries.

- [ ] **Step 4: Fix the section numbering (spec §5.4)**

`SKILL.md:125-133` numbers the `issue.specs` sections with **Header as 1**, making *Related local docs* §8 and *Context Gaps* §9. `issue-specs-template.md:80,88` numbers them **§7 and §8**. They disagree today, before this work. Align on the template's numbering — it is the file actually written — and renumber `SKILL.md`'s Step 6 list to match. Then grep the whole plugin for `Section 8` / `§ 4` style cross-references and fix any that shifted.

- [ ] **Step 5: README, config.md and the version bump**

README lines 41, 78, 111 → the `active/` layout. `config.md`'s "A different plans directory" → the knob is a **root**, with both branches of the resolution rule and the note that all **four** plugins must share the value. `plugin.json` → `"version": "0.4.0"`.

- [ ] **Step 6: Verify the cross-plugin contract**

This is the finding that broke the first draft, so it gets its own assertion. On a fixture whose `GH-999` is already closed (from Task 6), run the revised Step 5 logic with a search term that appears **only** in `docs/plans/closed/GH-999/phases/PHASE-01-service.md` — a file that carries no header.

Expected: exactly one line in the output, `- GH-999 · <status> · closed <date> — matched: idempotency`, with the status read from `master-plan.md`. Not zero lines (skipped), not a full excerpt entry, not one line per matching file.

Then delete the header from `master-plan.md` and re-run: the line must still appear, with status `unknown`.

- [ ] **Step 7: Commit**

```bash
git add plugins/create-master-plan
git commit -m "create-master-plan 0.4.0: active/closed layout, collapsed closed-plan reporting, section renumbering"
```

---

### Task 8: `decompose-plan` and `plan-review` path examples

**Files:**
- Modify: `plugins/decompose-plan/README.md:114,126,138,172` and any `docs/plans/GH-` in its `skills/**`
- Modify: `plugins/plan-review/README.md:121-122,169-170` and any in its `skills/**`
- Modify: both `.claude-plugin/plugin.json` (patch bumps: 0.3.5 → 0.3.6, 0.3.2 → 0.3.3)

**Interfaces:**
- Consumes: nothing. These plugins take the plan folder as an argument and their behaviour is unchanged.
- Produces: examples that point at paths which will exist.

- [ ] **Step 1: Rewrite the examples**

Every `docs/plans/GH-412` becomes `docs/plans/active/GH-412`. Behaviour text stays untouched — neither skill gains a layout rule, because neither needs one.

- [ ] **Step 2: Bump both manifests**

`decompose-plan` 0.3.5 → 0.3.6, `plan-review` 0.3.2 → 0.3.3.

- [ ] **Step 3: Verify**

Run: `grep -rn 'docs/plans/GH-' plugins/decompose-plan plugins/plan-review`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add plugins/decompose-plan plugins/plan-review
git commit -m "decompose-plan, plan-review: point examples at the active/ layout"
```

---

### Task 9: `MANUAL.html` — five steps become six

The largest single piece of work here, and the one to scope before starting rather than discover mid-edit.

**Files:**
- Modify: `plugins/create-master-plan/MANUAL.html` (the edit master), then copy to the other three
- Create: `plugins/close-master-plan/MANUAL.html` (the copy)

**Interfaces:**
- Consumes: the finished skill from Task 6 — the manual documents behaviour that must already exist.
- Produces: one file, four byte-identical copies.

- [ ] **Step 1: Inventory before editing**

Run: `grep -n -iE 'five step|all five|docs/plans|three plugin' plugins/create-master-plan/MANUAL.html`
Expected: at minimum line 144 (`<h2 id="glance">The five steps</h2>`), line 605 (`Run all five steps`), and lines 155, 211, 585 (`docs/plans` paths). Record the full list before touching anything — the count drives whether this task needs splitting.

- [ ] **Step 2: Edit the master copy**

Five steps → six throughout, including the `id="glance"` heading and its list; the three `docs/plans` paths → `docs/plans/active/`; the section on the plans directory (line 585) → the root rule; a new section for step 6 mirroring the others' structure.

- [ ] **Step 3: Sync all four**

```bash
for p in decompose-plan plan-review close-master-plan; do
  cp plugins/create-master-plan/MANUAL.html "plugins/$p/MANUAL.html"
done
```

- [ ] **Step 4: Verify**

Run: `md5sum plugins/*/MANUAL.html | awk '{print $1}' | sort -u | wc -l`
Expected: `1`.

Run: `grep -ic 'five step' plugins/create-master-plan/MANUAL.html`
Expected: `0`.

- [ ] **Step 5: Commit**

```bash
git add plugins/*/MANUAL.html
git commit -m "MANUAL.html: six steps, active/closed layout, synced across four plugins"
```

---

### Task 10: Repo surface — README and marketplace

**Files:**
- Modify: `README.md` (plugin table, line 60 heading and its paragraph, the workflow diagram at lines 63-72, the version column)
- Modify: `.claude-plugin/marketplace.json`

**Interfaces:**
- Consumes: the version numbers now in all four manifests.
- Produces: the catalogue entry that makes the plugin installable.

- [ ] **Step 1: Add the marketplace entry**

A fourth object in `plugins[]`, `"source": "./plugins/close-master-plan"`, description matching the manifest's.

- [ ] **Step 2: Update the README table**

New row for `close-master-plan` (Step 6), **and correct the stale versions**: the table says 0.3.3/0.3.4/0.3.1; after Tasks 7 and 8 the manifests say 0.4.0/0.3.6/0.3.3.

- [ ] **Step 3: Rewrite "The three are one workflow"**

Line 60's heading and paragraph become four, and the ASCII flow diagram at lines 63-72 gains the closing step:

```
/create-master-plan 412                   →  issue.specs · master-plan.md
        ↓  new conversation
/decompose-plan docs/plans/active/GH-412  →  phases/ · tasks.md · execute-plan.md · handoff.md
        ↓  new conversation
paste the Coordinator Prompt              →  one agent per phase, a round at a time
        ↓  optional
/plan-implementation-review               →  a prompt for a fresh reviewer, code against plan
        ↓
/close-master-plan                        →  reconciled · distilled · archived to closed/
```

- [ ] **Step 4: Verify**

Run: `"$SCRATCH/check.sh" "$PWD"`
Expected: assertions 1, 2 and 6 now pass. Only TUTORIAL-related failures from assertion 6 may remain, and Task 11 clears those.

- [ ] **Step 5: Commit**

```bash
git add README.md .claude-plugin/marketplace.json
git commit -m "marketplace: publish close-master-plan, four-step workflow, correct version table"
```

---

### Task 11: `TUTORIAL.md` — Part 6 and the path sweep

**Files:**
- Modify: `docs/master-plan-pack/TUTORIAL.md` (line 3, line 86, 22 path refs, the cheat sheet at ~line 990, new Part 6 after line 967)

**Interfaces:**
- Consumes: the finished behaviour from Tasks 6 and 7 — the tutorial is a worked example, so every command it shows must work.
- Produces: the pack's single end-to-end narrative, now closing the loop.

- [ ] **Step 1: The sweep**

Line 3 "Three plugins" → four, with the new link. Line 86 "§0.1 · The marketplace and the three plugins" → four. All 22 `docs/plans/GH-412` → `docs/plans/active/GH-412`, except any that are explicitly about the legacy layout.

- [ ] **Step 2: Retitle Part 5 and write Part 6**

Today's "Part 5 · Close out" (line 959) is three manual bullets. It becomes the introduction to `/close-master-plan`: the same three things, now a command, followed by what the command asks you for (status, the mapping table, the placeholder fills, the rule approvals) and what it prints.

Part 6 must cover, because these are the day-to-day traps: run it **after** the reviews; a dirty tree stops it; `abandoned` needs the cherry-pick; squash-merge invalidates the SHAs and the PR number is the durable pointer; and there is no reopen.

- [ ] **Step 3: Write the sequence diagram**

Mirroring the existing ones in style. **No `;` and no `<...>` anywhere in a label or note** — a `;` is a mermaid statement separator and silently truncates the diagram.

```
sequenceDiagram
    autonumber
    participant U as You
    participant S as /close-master-plan
    participant F as docs/plans/active/GH-412/
    participant R as .claude/rules/

    U->>S: /close-master-plan
    S->>F: read tasks.md, handoff.md, master-plan.md
    S-->>U: proposed phase to commit mapping
    U-->>S: confirmed
    S->>F: SHAs, Final Summary, filled handoff
    S-->>U: candidate rules, one at a time
    U-->>S: approve or decline each
    S->>R: write only what was approved
    S->>F: stamp STATUS on the three files
    S->>F: git mv to docs/plans/closed/GH-412/
    S-->>U: the commit command, and stop
    Note over U,R: the skill never commits and never touches the worktree
```

- [ ] **Step 4: Render-check every new block**

```bash
cd "$SCRATCH" && PUPPETEER_SKIP_DOWNLOAD=true npm i @mermaid-js/mermaid-cli
echo '{"executablePath":"/usr/bin/google-chrome-stable","args":["--no-sandbox","--disable-dev-shm-usage"]}' > pptr.json
# extract each new ```mermaid block to its own .mmd file first, so a failure names the diagram
node node_modules/.bin/mmdc -p pptr.json -i block.mmd -o /tmp/out.svg
```

Expected: exit 0 for every block. Then feed one deliberately broken block (add a `;` inside a Note) and confirm exit 1 — a checker that never fails proves nothing.

- [ ] **Step 5: Update the cheat sheet**

Add the install line (`/plugin install close-master-plan@necofx`), the `/close-master-plan` per-ticket line in the right position — after the reviews, before finishing — and update the paths in the existing lines.

- [ ] **Step 6: Commit**

```bash
git add docs/master-plan-pack/TUTORIAL.md
git commit -m "TUTORIAL: Part 6 close-out, active/ layout, four plugins"
```

---

### Task 12: Final gate

**Files:** none created or modified unless the checks find something.

**Interfaces:**
- Consumes: everything.
- Produces: the go/no-go.

- [ ] **Step 1: Full consistency check**

Run: `"$SCRATCH/check.sh" "$PWD"`
Expected: `check.sh: all assertions passed`.

- [ ] **Step 2: Install smoke test**

Run: `claude plugin marketplace add ./` then `claude plugin list`
Expected: four plugins listed, `close-master-plan` among them at 0.1.0.

- [ ] **Step 3: End-to-end on a fresh fixture**

Rebuild `$SCRATCH/fx` from scratch and run the whole pack against it: `/close-master-plan` with no argument, all the way to the printed commit command. This catches anything the per-task fixtures missed by starting mid-flow.

- [ ] **Step 4: Spec reconciliation**

Re-read `docs/superpowers/specs/2026-08-18-close-master-plan-design.md` §7 (failure modes) and confirm every row has a corresponding instruction in the shipped `SKILL.md`. Any row without one is a gap to close before declaring done.

- [ ] **Step 5: Commit the spec and plan**

They were written but never committed. They are the reasoning behind the change and belong in the record.

```bash
git add docs/superpowers
git commit -m "docs: close-master-plan design and implementation plan"
```
