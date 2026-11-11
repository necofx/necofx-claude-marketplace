---
name: create-master-plan
description: This skill should be used to bootstrap a brand-new master implementation plan for a ticket. Takes a GitHub issue number by default (or a Jira/Linear key, a URL, or a pasted requirement), creates the plan folder under `docs/plans/`, fetches the issue plus its linked items, cited documents and attachments, exhaustively scans the local `docs/` folder for related material, writes everything into `issue.specs`, then conducts an embedded project-agnostic interview to fill remaining gaps before producing a `master-plan.md` shaped by `superpowers:writing-plans`. Tracker-agnostic via the `source-adapters.md` reference (GitHub by default, Jira and Linear as profiles) and tech-stack agnostic via `tech-stack-profiles.md`. Trigger when the user invokes `/create-master-plan` with a ticket argument, or asks to "create a master plan for issue 412", "bootstrap a plan from this ticket", "start a PRP for GH-XXX", or "build a master plan from this issue".
---

# Create Master Plan

## Overview

Bootstraps a brand-new master implementation plan from a ticket. Pulls together everything that lives outside the implementer's head — the ticket body, comments, linked issues, cited documents, attachments, related local documentation — and turns it into two artifacts in `<plans-root>/active/<TICKET-ID>/` (see `references/plan-layout.md`):

1. **`issue.specs`** — every piece of fetched context, in a stable 9-section structure.
2. **`master-plan.md`** — the writing-plans-shaped plan, produced AFTER an embedded interview fills the gaps in `issue.specs`.

The skill stops at `master-plan.md`. The user invokes `/decompose-plan` separately to phase the plan out.

**GitHub is the default source.** Jira, Linear and free-form input are profiles in `references/source-adapters.md`, selected by the detection ladder in Step 2. Nothing after Step 2 knows or cares which one ran.

## Inputs

- **`<ticket>`** (required): a GitHub issue number (`412`, `#412`, or an issue URL) by default; a tracker key like `ACME-1234`, a Linear or Atlassian URL, a path to a local document, or a pasted requirement. Step 2's ladder resolves which adapter applies. The derived `<TICKET-ID>` is validated against `^[A-Z]+-[0-9]+$` for tracker sources, or is a confirmed slug for free-form input.

## Workflow

Before Step 0, read `references/operator-notes.md` — the standing rules every step below assumes.

Follow these steps in order.

### Step 0 — Parse the argument

1. Read the argument. If absent, ask the user via `AskUserQuestion`.
2. If it's a URL, extract the identifying segment: the trailing `[A-Z]+-[0-9]+` for a tracker URL, or the issue number from a `github.com/<owner>/<repo>/issues/<N>` URL.
3. Derive `<TICKET-ID>`:
   - GitHub issue number `N` → `GH-<N>` (uppercased, satisfies the tracker shape).
   - A tracker key → used verbatim, uppercased, validated against `^[A-Z]+-[0-9]+$`.
   - Free-form text or a file → `slugify(title)` truncated to 40 characters, **confirmed with the user**.
4. If a tracker-shaped argument fails validation, ask the user to re-enter rather than guessing.

The full ladder — including what to do when two adapters both match — is in `references/source-adapters.md`, applied in Step 2. Step 0 only needs enough to name the folder.

### Step 0.5 — Fork into a worktree for this ticket

Apply `references/git-lifecycle.md`'s **creation side** in full. In short: `EnterWorktree` on the
ticket id, rename the harness's `worktree-<id>` branch to `feature/<id>` before anything is pushed,
and — only when the parent repository already has a `.codegraph/` directory — start `codegraph init`
in the background so the index is ready by the time the interview needs it.

**This runs before Step 1, and the ordering is load-bearing.** Step 1 creates the plan folder in
whatever tree the session is standing in. Fork afterwards and that folder sits in the main checkout
while the worktree — a clean checkout of the new branch — does not contain it, so every later step
writes into a tree the branch will never carry. Forking here means `issue.specs`, the interview and
`master-plan.md` are born inside the worktree.

The reference's escape table applies silently and is never turned into a question: already inside a
worktree, not a git repository, no `origin`, or a harness that cannot move the session's cwd all
mean *continue without forking and say so*, not *stop*. A repository with no `.codegraph/` gets no
index — that standing rule is untouched.

### Step 1 — Folder setup (overwrite-aware)

1. Resolve the project root (nearest ancestor of CWD with a `.git` directory).
2. Resolve `<plans-root>` per `references/plan-layout.md`, then compute `<plan-folder> = <plans-root>/active/<TICKET-ID>/`.
3. If `<plans-root>/<TICKET-ID>/` already exists in the flat legacy layout (no `active/`/`closed/` split), use it unchanged as `<plan-folder>` instead of creating one under `active/`, and note in the Step 9 run summary that the project is on the old layout. This skill never migrates a project from the flat layout to `active/`/`closed/` on its own — that migration, if it happens, is `close-master-plan`'s business.
4. If `<plan-folder>` does NOT exist, create it (with `attachments/` lazily created on first write in Step 3).
5. If `<plan-folder>` exists and contains files, list them and use `AskUserQuestion` with three options:
   - **Overwrite specific files** (skill picks: typically `issue.specs` and `master-plan.md`; preserves `attachments/`, `phases/`, `tasks.md`, etc.)
   - **Merge** (skill renames `issue.specs` → `issue.specs.bak.<timestamp>` and `master-plan.md` → `master-plan.md.bak.<timestamp>`, then writes fresh files alongside)
   - **Abort** (skill stops with a one-paragraph message describing what's already there)

Do NOT proceed past this step if the user chose Abort.

### Step 2 — Pick the source adapter and fetch the ticket

Read `references/source-adapters.md` and apply its detection ladder. **First match wins, and ambiguity never guesses — it asks.** GitHub is the default; Jira, Linear, file and free-form are profiles in that file.

Run the adapter's fetch and normalise the result into the single intermediate record the reference describes: `id`, `title`, `body`, `state`, `author`, `assignees`, `labels`, `components`, `priority`, `due`, `parent`, `comments[]`, `links[]`, `documents[]`, `attachments[]`, `source_kind`, `fetched_at`.

**Everything after this step reads that record, not the tracker.** Steps 3–9 must never branch on `source_kind` except where `issue.specs` reports provenance — that is what lets the same skill serve a GitHub issue, a Jira ticket and a pasted paragraph without a special case at each stage.

Fields the source cannot supply are left empty and recorded for Context Gaps. An empty `components` on GitHub is data, not a defect; inventing an equivalent out of labels is worse than the gap.

If the adapter's fetch fails outright (auth, not-found), stop and report it to the user. Do NOT proceed — a dossier built on a failed fetch is worse than no dossier, because it looks complete.

### Step 3 — Download attachments

For each entry in the record's `attachments[]`:

1. Ensure `<plan-folder>/attachments/` exists (mkdir if not).
2. Target path: `<plan-folder>/attachments/<sanitized-original-filename>`. Sanitize by replacing path separators and any character not in `[A-Za-z0-9._\- ]` with `_`.
3. Download it the way the chosen adapter specifies — a plain fetch for GitHub's `user-attachments` URLs, the MCP `fetch` tool or an authenticated `curl` for Jira. If the download needs a credential you do not have, **write the command out for the user and continue without blocking the rest of the workflow.**
4. **Read the images.** You can see them, and a screenshot in a bug report is often the only place the actual behaviour is described.
5. Record outcome (downloaded / skipped + reason) for the `issue.specs § Attachments` section in Step 6.

If any attachment download fails, record the failure in the Context Gaps section but DO NOT abort the run.

### Step 4 — Fetch the linked items and the cited documents

Cap context: **≤ 5 linked items, ≤ 3 documents**. These caps keep the fetched context useful without drowning the interview; raise them only when your tickets are unusually sparse.

**4a. Linked items.** Fetch each entry in the record's `links[]` using the adapter's recipe in `references/source-adapters.md` — enough for a one-paragraph summary each: title, state, and the part of its body that bears on this ticket.

Priority order (stop after 5): parent epic > "blocks" / "is blocked by" > "relates to" > sub-tasks > everything else. Record the relation type alongside each one.

**On GitHub, a referenced pull request is the highest-value fetch in this step.** Its file list tells you what already landed, which is how "this ticket is gap-closure, not a build" gets discovered cheaply instead of by grepping blind. Carry that file list forward — the interview and the master plan both need it.

**4b. Documents.** Fetch every document in the record's `documents[]` — a Confluence page, an RFC, a design doc, a linked README — and **quote its content into `issue.specs` rather than linking it.** The whole point of the dossier is that nobody has to return to the tracker, and a link is a return to the tracker.

Where the adapter supports a search top-up (Jira's CQL, for instance) and fewer than 3 documents were found, use it with the ticket's distinctive summary nouns and any component name, keeping the top 3 by title match.

If any document fetch fails (auth, missing space, deleted page), record the failure in Context Gaps and continue — never block the run.

### Step 5 — Scan the local `docs/` folder EXHAUSTIVELY

**Read `references/docs-scan.md` now and follow it.** No folder under `docs/` is excluded and no cap applies: every `docs/**/*.md` is grepped for the ticket id, the component names and the summary's distinctive nouns. Matches under `<plans-root>/closed/` are collapsed to one line per plan folder; everything else is reported in full.

### Step 5.5 — Detect the project's tech stack(s)

Read `references/tech-stack-profiles.md` for the detection precedence and the field shape of each profile. Apply it now so the interview in Step 7 and the master plan in Step 8 use the right build/test commands, the right coding-rules files, and the right specialist agent recommendations.

Precedence:
1. The ticket may name the stack explicitly in the body, components, or labels — use that first.
2. Otherwise apply root-marker detection from the working directory (`*.sln`, `package.json`, `*.dproj`, …).
3. If the project mixes stacks, record the layer→stack mapping so the master plan's Tech Stack section can list each.
4. If ambiguous, ask the user via `AskUserQuestion` at the START of the interview (Step 7) — phrased as the first question.

Record the resolved stack(s). This drives the master-plan template's Pre-flight Checklist, Test Configuration, Validation & Testing, and Owner-skill recommendations.

### Step 6 — Write `issue.specs`

Write `<plan-folder>/issue.specs` using the structure defined in `references/issue-specs-template.md`. Sections, in order (the Header is unnumbered; the numbering below matches `issue-specs-template.md` exactly):

- **Header** — `<TICKET-ID>` · title · state · type · priority · author · assignee · source (`github` | `jira` | `linear` | `file` | `free-form`) · fetched-at (`YYYY-MM-DD HH:MM`) · plan folder path
1. **Description** — the raw ticket body (markdown), verbatim
2. **Acceptance Criteria** — extracted verbatim if the description has an `Acceptance Criteria` / `AC` / `Definition of Done` / `DoD` heading (case-insensitive); otherwise `_(none in the ticket — to be filled by the interview)_`
3. **Comments** — chronological, each as a sub-heading `### <author> · <YYYY-MM-DD HH:MM>` followed by the body
4. **Linked issues** — each as `### <id> · <relation> · <state>` then a 1-paragraph summary. On GitHub, a merged PR's file list belongs here.
5. **Referenced documents** — each as `### <title>` with the URL and either the full body (if <2k chars) or the most relevant excerpt
6. **Attachments** — table: filename · size · mime · local path · download status
7. **Related local docs** — for every match from Step 5 outside `<plans-root>/closed/`: `- <relative-path> — matched: <terms> — excerpt: <one line>`. May be long; do not truncate. Followed by a `### Closed plans` subsection holding the collapsed one-line-per-folder entries from Step 5's `closed/` partition, kept separate from the full entries above it.
8. **Context Gaps** — every failure recorded earlier (attachment download fails, document auth fails, fields the source cannot supply, an empty thread on free-form input). Empty bullet `_(none)_` if all clean.

Do NOT write Section 9+ — Interview Notes (Section 9) is appended in Step 7, and the master plan is a separate file (`master-plan.md`) written in Step 8.

### Step 7 — Run the embedded interview

The interview protocol lives in `references/interview-protocol.md` (adapted from the project's `.claude/commands/interview.md`). Read it once at the start of this step and follow it.

Key adaptations from the original:
- The spec being interviewed is `<plan-folder>/issue.specs` (not an arbitrary file).
- The output of the interview is `<plan-folder>/master-plan.md` (written in Step 8), not `{name}_spec.md`.
- Q&A rounds are captured back into `issue.specs` as a new `## Interview Notes` section appended at the bottom, so future runs of this skill can see what was discussed without re-fetching context.

Coverage areas (as listed in `references/interview-protocol.md`): technical implementation, UI/UX, edge cases, tradeoffs, acceptance criteria, testing strategy, validation requirements. Use `AskUserQuestion` for every clarifying question. Continue until the user signals the spec is complete.

Before exiting the interview, summarise the proposed master-plan outline and ask the user to confirm via `AskUserQuestion` (one final yes/no/edit).

### Step 8 — Generate `master-plan.md`

Invoke `Skill(skill="superpowers:using-superpowers")`, then `Skill(skill="superpowers:writing-plans")`. Then **read `references/master-plan-generation.md` now and follow it**: which parts of writing-plans apply to a plan that will be decomposed rather than executed (decisions not code, Review Focus, the proportion check — and never its Execution Handoff), what the plan MUST contain, and the fallback when writing-plans is unavailable.

### Step 9 — Report back

Reply to the user with, in this order:

1. **Where the session is standing**: the worktree path and the branch name, or an explicit "not forked" plus the reason from `references/git-lifecycle.md`'s escape table. A session that silently changed directory is the one thing the user must never have to discover for themselves.
2. **Which CodeGraph index is in play**: a worktree-local one (with its size), the parent repository's (with the caveat that it reflects a different working tree), or none.
3. The plan folder absolute path.
4. Files written: `issue.specs` (size), `master-plan.md` (size), attachments downloaded vs skipped (count + names).
5. Context fetched: the adapter used, 1 ticket + N linked items + M documents.
6. Related local docs: count + the top 5 by match strength.
7. Interview rounds: count of `AskUserQuestion` calls.
8. Suggested next step: `Run /decompose-plan <plans-root>/active/<TICKET-ID>` to phase out the master plan into atomic phases.

Do NOT begin decomposition in this conversation — that's the job of the `decompose-plan` skill, which the user invokes separately.

## Resources

### `references/interview-protocol.md`
The project-agnostic interview protocol. Read once at the start of Step 7. Sets the rules for the interview (coverage areas, AskUserQuestion-only, depth, summary-before-write) and points at `references/master-plan-template.md` for the output shape and `references/tech-stack-profiles.md` for stack-specific tailoring.

### `references/issue-specs-template.md`
The canonical 9-section structure for `issue.specs`. Read once at the start of Step 6 and apply every section even when empty (use `_(none)_` placeholders). Project-agnostic.

### `references/tech-stack-profiles.md`
Profiles for .NET, Blazor, React, Delphi, and Java/JVM (build/test commands, test framework, assertion + mocking conventions, code-style notes, conventions location, commit format, root-marker detection). Read in Step 5.5 to pick the right stack(s); referenced again in Steps 7 and 8 whenever a template needs a stack-specific value substituted in. Add new profiles here when working on a different stack rather than hardcoding values in the templates.

### `references/master-plan-template.md`
The writing-plans-shaped skeleton for `master-plan.md`. Read once at the start of Step 8. Sections: Context, Pre-flight Checklist, Why, Out of scope, Technical Requirements, Implementation Outline (with Key interfaces), Test Configuration, Validation & Testing, Acceptance Criteria, Review Focus, Success Criteria, Attachments, Related Docs. Stack-specific values come from the matching profile in `tech-stack-profiles.md`.

### `references/git-lifecycle.md`
The git contract this plugin shares with `close-master-plan`: the four states a ticket passes through (`unforked` → `forked` → `proposed` → `clean`), how each is *derived* from git and `gh` rather than stored, the creation side this skill applies in Step 0.5 (fork, branch rename, CodeGraph policy, escape table), and the two-phase close the other skill applies. Read once at the start of Step 0.5. **Byte-identical to `close-master-plan`'s copy — edit one, copy it to the other.**

## Notes for the skill operator

Standing rules for the whole run — CodeGraph use, MCP naming, read-only trackers, git policy, no customer data, absolute dates, stack agnosticism — live in `references/operator-notes.md`. Read it once before Step 0.
