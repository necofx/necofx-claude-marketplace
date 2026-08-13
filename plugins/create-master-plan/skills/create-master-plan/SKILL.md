---
name: create-master-plan
description: This skill should be used to bootstrap a brand-new master implementation plan for a Jira issue. Takes a Jira issue key (e.g. {{TICKET_PREFIX}}-1234) as input, creates the plan folder under `{{PLANS_DIR}}/`, fetches the issue plus linked Jira issues, Confluence pages and attachments, exhaustively scans the local `docs/` folder for related material, writes everything into `issue.specs`, then conducts an embedded project-agnostic interview to fill remaining gaps before producing a `master-plan.md` shaped by `superpowers:writing-plans`. Tech-stack agnostic — supports .NET, Blazor, React, and Delphi out of the box via the `tech-stack-profiles.md` reference, and is easily extended for other stacks. Trigger when the user invokes `/create-master-plan` with a Jira key argument, or asks to "create a master plan for {{TICKET_PREFIX}}-XXXX", "bootstrap a plan from this Jira ticket", "start a PRP for {{TICKET_PREFIX}}-XXXX", or "build a master plan from Jira".
---

# Create Master Plan

## Overview

Bootstraps a brand-new master implementation plan from a Jira ticket. Pulls together everything that lives outside the implementer's head — Jira description, comments, linked issues, Confluence pages, attachments, related local documentation — and turns it into two artifacts in `{{PLANS_DIR}}/<JIRA-KEY>/`:

1. **`issue.specs`** — every piece of fetched context, in a stable 9-section structure.
2. **`master-plan.md`** — the writing-plans-shaped plan, produced AFTER an embedded interview fills the gaps in `issue.specs`.

The skill stops at `master-plan.md`. The user invokes `/decompose-plan` separately to phase the plan out.

## Inputs

- **`<jira-key>`** (required): bare key like `{{TICKET_PREFIX}}-1234`, or a full Atlassian URL (e.g. `https://{{JIRA_CLOUD_ID}}/browse/{{TICKET_PREFIX}}-1234`). Validated against `^[A-Z]+-[0-9]+$` after URL parsing.

## Workflow

Follow these steps in order.

### Step 0 — Parse the argument

1. Read the argument. If absent, ask the user via `AskUserQuestion`.
2. If it's a URL, extract the trailing `[A-Z]+-[0-9]+` segment.
3. Validate the result against `^[A-Z]+-[0-9]+$`. If invalid, ask the user to re-enter.
4. Uppercase the result. Save as `<JIRA-KEY>`.

### Step 1 — Folder setup (overwrite-aware)

1. Resolve the project root (nearest ancestor of CWD with a `.git` directory).
2. Compute `<plan-folder> = <project-root>/{{PLANS_DIR}}/<JIRA-KEY>/`.
3. If `<plan-folder>` does NOT exist, create it (with `attachments/` lazily created on first write in Step 3).
4. If `<plan-folder>` exists and contains files, list them and use `AskUserQuestion` with three options:
   - **Overwrite specific files** (skill picks: typically `issue.specs` and `master-plan.md`; preserves `attachments/`, `phases/`, `tasks.md`, etc.)
   - **Merge** (skill renames `issue.specs` → `issue.specs.bak.<timestamp>` and `master-plan.md` → `master-plan.md.bak.<timestamp>`, then writes fresh files alongside)
   - **Abort** (skill stops with a one-paragraph message describing what's already there)

Do NOT proceed past this step if the user chose Abort.

### Step 2 — Fetch the Jira issue

Use the Atlassian MCP. Tool names vary by MCP server configuration; the canonical operation is `getJiraIssue`. If unavailable, search for it via `ToolSearch` query `+jira getJiraIssue` and load it.

Call the equivalent of:

```
getJiraIssue
  cloudId: {{JIRA_CLOUD_ID}}
  issueIdOrKey: <JIRA-KEY>
  responseContentFormat: markdown
  fields: ["summary", "description", "status", "labels", "components",
           "comment", "reporter", "assignee", "issuetype", "parent",
           "priority", "duedate", "fixVersions", "issuelinks", "attachment"]
```

Capture for downstream use:
- summary, description (full markdown), status name
- reporter (accountId + displayName), assignee (accountId + displayName)
- labels, components, fixVersions, priority, duedate, issuetype
- parent (epic) if present
- every `issuelinks` entry (type + outward/inward issue keys)
- every comment (chronological — author + created date + body)
- every attachment (filename, mime type, size, content URL)

If Jira returns an error (auth, not-found), stop and report it to the user. Do NOT proceed.

### Step 3 — Download attachments

For each attachment captured in Step 2:

1. Ensure `<plan-folder>/attachments/` exists (mkdir if not).
2. Target path: `<plan-folder>/attachments/<sanitized-original-filename>`. Sanitize by replacing path separators and any character not in `[A-Za-z0-9._\- ]` with `_`.
3. Try in this order:
   - If the Atlassian MCP `fetch` tool (or any equivalent that handles authenticated Atlassian binary downloads) is available, use it.
   - Otherwise, write a downloadable shell command to the user:
     ```powershell
     # Run this to fetch attachment <filename>:
     curl -L -o "<target-path>" -u "<email>:<atlassian-api-token>" "<content-url>"
     ```
     and continue without blocking the rest of the workflow.
4. Record outcome (downloaded / skipped + reason) for the `issue.specs § Attachments` section in Step 6.

If any attachment download fails, record the failure in the Context Gaps section but DO NOT abort the run.

### Step 4 — Fetch linked Jira issues and Confluence pages

Cap context: **≤ 5 linked Jira issues, ≤ 3 Confluence pages**. These caps keep the fetched context useful without drowning the interview; raise them only when your tickets are unusually sparse.

**4a. Linked Jira issues.** For each entry in `issuelinks` plus the `parent` (epic) if present, fetch with:

```
getJiraIssue
  cloudId: {{JIRA_CLOUD_ID}}
  issueIdOrKey: <linked key>
  responseContentFormat: markdown
  fields: ["summary", "description", "status", "issuetype"]
```

Priority order (stop after 5): parent epic > "blocks" / "is blocked by" > "relates to" > sub-tasks > everything else. Record the link type alongside each linked issue.

**4b. Confluence pages.**
1. Extract every `https://*.atlassian.net/wiki/...` URL from the ticket's description and comments.
2. For each, parse the page ID from the URL and call:
   ```
   getConfluencePage
     cloudId: {{JIRA_CLOUD_ID}}
     pageId: <page id>
   ```
3. If fewer than 3 URLs were found, top up via CQL using the ticket's summary nouns and any component name:
   ```
   searchConfluenceUsingCql
     cloudId: {{JIRA_CLOUD_ID}}
     cql: text ~ "<keywords>" AND type = page
     limit: 5
   ```
   Keep only the top 3 by title-match.

If any Confluence call fails (auth, missing space, deleted page), record the failure in Context Gaps and continue — never block the run.

### Step 5 — Scan the local `docs/` folder EXHAUSTIVELY

Per the user's explicit direction, no folder is excluded and no cap is applied beyond glob limits.

1. Build the haystack of search terms:
   - The Jira key itself (e.g. `{{TICKET_PREFIX}}-1234`).
   - Every component name from Step 2.
   - The most distinctive nouns from the ticket summary (skip stopwords).
2. Glob `<project-root>/docs/**/*.md` (no exclusion, no depth cap).
3. For each candidate file, run a case-insensitive Grep for any of the haystack terms. A single hit is enough.
4. For each match, record:
   - relative path from `<project-root>`
   - which terms matched
   - a 1–2 line excerpt around the first match (use Grep with `-B 1 -A 1`)

This produces a potentially long list — that's expected. Section 8 of `issue.specs` is allowed to grow.

### Step 5.5 — Detect the project's tech stack(s)

Read `references/tech-stack-profiles.md` for the detection precedence and the field shape of each profile. Apply it now so the interview in Step 7 and the master plan in Step 8 use the right build/test commands, the right coding-rules files, and the right specialist agent recommendations.

Precedence:
1. The Jira ticket may name the stack explicitly in the description, components, or labels — use that first.
2. Otherwise apply root-marker detection from the working directory (`*.sln`, `package.json`, `*.dproj`, …).
3. If the project mixes stacks, record the layer→stack mapping so the master plan's Tech Stack section can list each.
4. If ambiguous, ask the user via `AskUserQuestion` at the START of the interview (Step 7) — phrased as the first question.

Record the resolved stack(s). This drives the master-plan template's Pre-flight Checklist, Test Configuration, Validation & Testing, and Owner-skill recommendations.

### Step 6 — Write `issue.specs`

Write `<plan-folder>/issue.specs` using the structure defined in `references/issue-specs-template.md`. Sections, in order:

1. **Header** — `<JIRA-KEY>` · summary · status · issuetype · priority · reporter · assignee · fetched-at (`YYYY-MM-DD HH:MM`) · plan folder path
2. **Description** — raw Jira description (markdown)
3. **Acceptance Criteria** — extracted verbatim if the description has an `Acceptance Criteria` / `AC` / `Definition of Done` / `DoD` heading (case-insensitive); otherwise `_(none in the ticket — to be filled by the interview)_`
4. **Comments** — chronological, each as a sub-heading `### <author> · <YYYY-MM-DD HH:MM>` followed by the body
5. **Linked Jira issues** — each as `### <KEY> · <link type> · <status>` then a 1-paragraph summary
6. **Confluence pages** — each as `### <title>` with the URL and either the full body (if <2k chars) or the most relevant excerpt
7. **Attachments** — table: filename · size · mime · local path · download status
8. **Related local docs** — for every match from Step 5: `- <relative-path> — matched: <terms> — excerpt: <one line>`. May be long; do not truncate.
9. **Context Gaps** — every failure recorded earlier (attachment download fails, Confluence auth fails, missing fields). Empty bullet `_(none)_` if all clean.

Do NOT write Sections 10+ — those are interview notes and the plan, written in later steps.

### Step 7 — Run the embedded interview

The interview protocol lives in `references/interview-protocol.md` (adapted from the project's `.claude/commands/interview.md`). Read it once at the start of this step and follow it.

Key adaptations from the original:
- The spec being interviewed is `<plan-folder>/issue.specs` (not an arbitrary file).
- The output of the interview is `<plan-folder>/master-plan.md` (written in Step 8), not `{name}_spec.md`.
- Q&A rounds are captured back into `issue.specs` as a new `## Interview Notes` section appended at the bottom, so future runs of this skill can see what was discussed without re-fetching context.

Coverage areas (as listed in `references/interview-protocol.md`): technical implementation, UI/UX, edge cases, tradeoffs, acceptance criteria, testing strategy, validation requirements. Use `AskUserQuestion` for every clarifying question. Continue until the user signals the spec is complete.

Before exiting the interview, summarise the proposed master-plan outline and ask the user to confirm via `AskUserQuestion` (one final yes/no/edit).

### Step 8 — Generate `master-plan.md`

Before writing, invoke (in this order):

1. `Skill(skill="superpowers:using-superpowers")` — establishes skill discipline.
2. `Skill(skill="superpowers:writing-plans")` — informs the master-plan structure.

Then write `<plan-folder>/master-plan.md` using the skeleton from `references/master-plan-template.md`. Fill every section based on the combined content of `issue.specs` (Sections 1–9), the captured Interview Notes (`## Interview Notes` at the bottom of `issue.specs`), and the resolved tech-stack(s) from Step 5.5.

Substitute the stack-specific build commands, test commands, owner-skill recommendations, and coding-rules file paths from the matching profile(s) in `references/tech-stack-profiles.md`. For mixed-stack projects, the master plan's Tech Stack section lists each stack with the layer it covers; each layer's Implementation Outline references the right profile.

The master plan MUST:
- Open with a Context block linking back to `issue.specs` so a fresh reader has the full trail.
- List every attachment under `attachments/` with a one-line description each.
- Be self-contained enough that a future `/decompose-plan` run can extract atomic phases from it without re-reading Jira.
- Use the project's own coding-rules / log-hygiene conventions when discussing implementation. When the project enforces no-internal-context-in-log-strings (e.g. via `.claude/rules/logging.md`), call that constraint out in the Technical Requirements section. Plan-level prose can name the ticket key freely; only the *recommendations for code* must be hygienic.

If `superpowers:writing-plans` is unavailable in the current environment, fall back to the skeleton in `references/master-plan-template.md` and note the fallback in the Skill operator notes section of the final file.

### Step 9 — Report back

Reply to the user with, in this order:

1. The plan folder absolute path.
2. Files written: `issue.specs` (size), `master-plan.md` (size), attachments downloaded vs skipped (count + names).
3. Jira context fetched: 1 issue + N linked + M Confluence pages.
4. Related local docs: count + the top 5 by match strength.
5. Interview rounds: count of `AskUserQuestion` calls.
6. Suggested next step: `Run /decompose-plan {{PLANS_DIR}}/<JIRA-KEY>` to phase out the master plan into atomic phases.

Do NOT begin decomposition in this conversation — that's the job of the `decompose-plan` skill, which the user invokes separately.

## Resources

### `references/interview-protocol.md`
The project-agnostic interview protocol. Read once at the start of Step 7. Sets the rules for the interview (coverage areas, AskUserQuestion-only, depth, summary-before-write) and points at `references/master-plan-template.md` for the output shape and `references/tech-stack-profiles.md` for stack-specific tailoring.

### `references/issue-specs-template.md`
The canonical 9-section structure for `issue.specs`. Read once at the start of Step 6 and apply every section even when empty (use `_(none)_` placeholders). Project-agnostic.

### `references/tech-stack-profiles.md`
Profiles for .NET, Blazor, React, and Delphi (build/test commands, test framework, assertion + mocking conventions, code-style notes, conventions location, commit format, root-marker detection). Read in Step 5.5 to pick the right stack(s); referenced again in Steps 7 and 8 whenever a template needs a stack-specific value substituted in. Add new profiles here when working on a different stack rather than hardcoding values in the templates.

### `references/master-plan-template.md`
The writing-plans-shaped skeleton for `master-plan.md`. Read once at the start of Step 8. Sections: Context, Pre-flight Checklist, Why, Out of scope, Technical Requirements, Implementation Outline, Test Configuration, Validation & Testing, Acceptance Criteria, Success Criteria, Attachments, Related Docs. Stack-specific values come from the matching profile in `tech-stack-profiles.md`.

## Notes for the skill operator

- **CodeGraph, when the repo has one.** If a `.codegraph/` directory exists at the repository root, ground the interview (Step 7) with `codegraph_explore` (MCP) or `codegraph explore "<symbols or question>"` (shell) instead of a `Glob`/`Grep`/`Read` loop. One call returns the relevant symbols' verbatim line-numbered source, the call paths between them, and what depends on them — far cheaper in tokens than reading every file a text search matched, and it follows dynamic dispatch that a text search cannot. This does NOT replace Step 5's `docs/**/*.md` scan: CodeGraph indexes code, not prose. If there is no `.codegraph/` directory, use the ordinary tools and do NOT index the repository yourself — that is the user's decision.
- **MCP tool naming**: the Atlassian MCP server name and tool-name prefix varies by install. The canonical operations are `getJiraIssue`, `getConfluencePage`, `searchConfluenceUsingCql`, `searchJiraIssuesUsingJql`, `lookupJiraAccountId`, and `fetch`. Use `ToolSearch` to locate them at runtime; do not hardcode UUID prefixes in this file.
- **Read-only intent for Jira**: this skill never writes to Jira (no comments, no transitions, no label edits). Read-only fetches only.
- **Git policy is project-dependent**. Some projects make Claude Code read-only for git (e.g. via `.claude/rules/git-workflow.md`); others let teammates commit themselves. Default to read-only when the project has no explicit policy — the user commits the produced files when they choose. Check before the report-back in Step 9.
- **No customer data in summaries**: when paraphrasing the Jira description for the report-back in Step 9, never include real customer names, identifiers, or credentials.
- **Skip if a recent plan exists**: when Step 1 finds an existing `master-plan.md` modified within the last 24 hours, mention this in the AskUserQuestion options so the user can choose to bail out without re-fetching everything.
- **Date format**: always absolute (`YYYY-MM-DD HH:MM`), never relative.
- **Tech-stack agnostic**: this skill works on .NET, Blazor, React, and Delphi projects out of the box (and any stack you add to `tech-stack-profiles.md`). The interview and master-plan templates pull stack-specific values from there; do NOT hardcode build/test commands or coding-rules paths in the skill itself.
- **Mixed-stack projects**: a Jira ticket may span layers (e.g. add a REST endpoint + the React UI that calls it). Record all relevant stacks during Step 5.5; the master plan's Implementation Outline groups work by layer with the matching stack profile.
