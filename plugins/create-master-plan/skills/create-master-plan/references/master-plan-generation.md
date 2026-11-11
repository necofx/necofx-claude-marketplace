# Step 8 — Generate master-plan.md

_Moved out of `SKILL.md` so it survives context compaction: the skill reads this file from disk at the step that needs it._

### Step 8 — Generate `master-plan.md`

`superpowers:using-superpowers` and `superpowers:writing-plans` are already loaded — SKILL.md's Step 8 stub invokes them before sending you here.

Write `<plan-folder>/master-plan.md` using the skeleton from `references/master-plan-template.md`. Fill every section based on the combined content of `issue.specs` (the Header and Sections 1–8), the captured Interview Notes (`## Interview Notes` at the bottom of `issue.specs`), and the resolved tech-stack(s) from Step 5.5.

Substitute the stack-specific build commands, test commands, owner-skill recommendations, and coding-rules file paths from the matching profile(s) in `references/tech-stack-profiles.md`. For mixed-stack projects, the master plan's Tech Stack section lists each stack with the layer it covers; each layer's Implementation Outline references the right profile.

The master plan MUST:
- Open with a Context block linking back to `issue.specs` so a fresh reader has the full trail.
- List every attachment under `attachments/` with a one-line description each.
- Be self-contained enough that a future `/decompose-plan` run can extract atomic phases from it without re-reading the tracker.
- Use the project's own coding-rules / log-hygiene conventions when discussing implementation. When the project enforces no-internal-context-in-log-strings (e.g. via `.claude/rules/logging.md`), call that constraint out in the Technical Requirements section. Plan-level prose can name the ticket key freely; only the *recommendations for code* must be hygienic.

Take from `superpowers:writing-plans` what fits a plan that will be decomposed, not executed:

- **Decisions, not code.** Record signatures, values from the spec, test names with their assertions, commands. A code body only for an algorithm those don't determine. Keep the `**Spec:**` line pointing at `issue.specs`.
- **Review Focus** is filled, or explicitly empty after checking.
- **Proportion check.** Compare the plan's length to `issue.specs`; a plan several times longer, dominated by code blocks, is a transcript — replace bodies with signatures and assertions.
- **Not its task breakdown or its Execution Handoff.** Phases are `/decompose-plan`'s job, and execution is `/execute-master-plan`'s: never offer Subagent-driven vs Native execution here.

If `superpowers:writing-plans` is unavailable in the current environment, fall back to the skeleton in `references/master-plan-template.md` and note the fallback in the final file's "Notes for the implementer" section.
