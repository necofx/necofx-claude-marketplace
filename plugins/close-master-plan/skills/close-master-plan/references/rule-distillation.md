# Rule distillation

What Step 6 does: read what the run actually learned, filter it hard, and let the user decide —
one candidate at a time — whether any of it earns a permanent place in `.claude/rules/`.

## Where to read from

Three sources, no others:

- `handoff.md` → *Key deviations from the original plan*. Written for reviewers, so it already
  states what the plan assumed, what the code does, and why — the raw material for a rule.
- `tasks.md` → `Decisions`. Justifications for dropped phases and other coordinator-level calls.
- `tasks.md` → `Coordination Notes`. Round summaries, cross-phase decisions, file-conflict
  resolutions, reassignments, scope adjustments.

Skip this step's reading entirely if the plan folder has no `tasks.md` (Step 4a applied) — there
is nothing decomposition-level to distil from a plan that was never phased out. `handoff.md` alone,
when present, is still fair game.

An empty section, an unfilled `_(to be filled)_` placeholder, or `_none yet_` is not a candidate —
skip it silently, the same way Step 5 treats a genuinely empty deviations section as fine.

## The durability test

Apply this one question to every line pulled from those sources:

> **Would this still be true on the next ticket, in a different area of the codebase?**

Not "was it correct" or "was it a good call" — both of the examples below were the right decision
for their own phase. The test is about scope, not quality:

- A statement about *this specific method, this specific ticket, this specific one-time collision*
  fails. It's true once. Writing it into `.claude/rules/` would tell a future agent working on
  unrelated code to imitate a decision that only made sense because of a coincidence that won't
  recur.
- A statement about *how this codebase names things, structures things, or handles a whole category
  of situation* passes. It holds regardless of which ticket or which file prompted the discovery.

If a candidate is ambiguous, phrase the test as a concrete follow-up question and ask the user
rather than guessing — "is this specific to the refund method, or does every method in this service
follow the same pattern?" The user's answer settles it, not a re-reading of the source text.

### Worked example — passes

> Phase 03 named the Helm value `partialRefunds.enabled` because Helm values are camelCase by
> chart convention, not the Java property's usual case.

Apply the test: the *next* phase that adds a Helm value — in a completely different chart, for a
completely different feature — faces the identical question, and the chart's naming convention
answers it the same way every time. This isn't about partial refunds; it's about how this repo's
Helm values are cased. It generalizes cleanly to a rule: **"Helm chart values use camelCase,
independent of the backing Java property's case."** Candidate.

### Worked example — fails

> Phase 01 reused an existing (previously unused) `amount` parameter that GH-388 had added, rather
> than introducing a second one.

Apply the test: this is a fact about one method, in one file, that happened to already have a
compatible parameter sitting unused from an earlier ticket. The next ticket won't find that same
unused parameter waiting — there's nothing here to reuse a decision *from*. It doesn't teach
"prefer existing parameters over new ones" as a general principle either; it's not a style
preference, it's a one-time observation that a coincidence existed and was exploited. It's a fact
about GH-388 and GH-999, not a fact about the codebase. **Not a candidate** — say so, rather than
writing it down or silently dropping it without comment. The user should see that this source was
read and rejected, not just that it was skipped.

## Which file to write to

Detect the stack the same way `create-master-plan`'s `references/tech-stack-profiles.md` does —
same precedence, don't reinvent it: master plan's `## Tech Stack` section first, then root markers
(`*.csproj`/`*.sln`, `package.json`, `*.dproj`, `build.gradle*`/`pom.xml`, …), then per-phase file
hints, then ask if still ambiguous. The matching stack points at the conventional file:
`.claude/rules/java.md`, `.claude/rules/dotnet.md`, `.claude/rules/react.md`, and so on. A mixed
project can produce candidates for more than one file — write each candidate to the file matching
the code it's about, not to whichever file the plan's primary stack suggests.

If `.claude/rules/` doesn't exist in the repo yet, offer to create it (and the one target file)
before writing the first approved candidate. If the user declines, skip this step entirely — do
not write rules files somewhere else instead, and do not silently drop the candidates either; note
in the close-out record that distillation was skipped by the user's choice.

## Output shape

For every candidate that survives the durability test:

1. Present it as a **concrete diff** against the target rules file — the exact lines to append (or
   amend), not a description of what the diff would contain. If the target file doesn't exist yet,
   the diff is against an empty file.
2. Ask for **explicit approval on that candidate alone**, via `AskUserQuestion` or an equivalent
   yes/no. Approving one candidate never bundles or implies approval of any other.
3. Only on a yes, apply that one diff to the file and record the file's path for Step 7.
4. On a no (or "skip"), move to the next candidate. The file is untouched — not even opened for
   writing.

**Nothing is ever written to `.claude/rules/` without an explicit per-candidate yes.** There is no
path in this step where the skill applies a diff on its own judgment, batches approvals, or treats
silence as consent — every write traces back to one candidate the user said yes to, individually.

**Zero rules is a valid and common outcome**, not a failure state and not something to apologize
for. Most runs turn up nothing that clears the durability bar — plenty of tickets are executed
exactly as planned, and plenty of deviations really are one-off. When every candidate is rejected,
or none of the sources produced a candidate in the first place, say plainly that no rules were
distilled and why (nothing passed the test / nothing to read from), and move on. That is the skill
working correctly, not it having failed to find something.

## What Step 7 needs back

The list of files actually written to in this step (after user approval), for the header's
`rules:` field — comma-separated paths, or `—` if the list is empty.
