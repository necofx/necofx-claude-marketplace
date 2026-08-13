# issue.specs — canonical structure

Every `issue.specs` produced by the `create-master-plan` skill MUST use this 9-section structure, in this order, even when a section is empty (use `_(none)_` placeholders).

Sections 10+ (Interview Notes, Plan) are appended later by Steps 7–8 of the parent skill. Do NOT write them in Step 6.

---

# {TICKET-ID} — {Title}

**Status:** {status}  
**Type:** {type}  
**Priority:** {priority}  
**Author:** {author}  
**Assignee:** {assignee}  
**Components:** {comma-separated, or `_(not supplied by this source)_`}  
**Labels:** {comma-separated}  
**Milestone / fix version:** {comma-separated, or `_(none)_`}  
**Due date:** {YYYY-MM-DD or `_(none)_`}  
**Parent:** {epic / tracking issue + title, or `_(none)_`}  
**Source:** {github | jira | linear | file | free-form}  
**Fetched at:** {YYYY-MM-DD HH:MM}  
**Plan folder:** `docs/plans/{TICKET-ID}/`

---

## 1. Description

{Raw ticket body — markdown, verbatim. Do NOT summarise. If empty, write `_(empty in the source — the interview will fill it)_`.}

## 2. Acceptance Criteria

{Verbatim extract of any section in the description whose heading matches `Acceptance Criteria`, `AC`, `Definition of Done`, or `DoD` (case-insensitive). If none, write `_(none in the ticket — to be filled by the interview)_`.}

## 3. Comments

{Chronological. For each comment:}

### {Author display name} · {YYYY-MM-DD HH:MM}

{Comment body, verbatim. Keep the original markdown formatting where possible.}

{If no comments: `_(no comments on this ticket)_`}

## 4. Linked issues

{For each linked item (up to 5, prioritised per SKILL.md Step 4a):}

### {id} · {relation} · {state}

**Title:** {title}

{1-paragraph summary of the linked item — focus on what's relevant to {TICKET-ID}, not the full text. For a MERGED pull request, list the files it touched: that is the cheapest evidence of what already landed.}

{If none: `_(no linked issues)_`}

## 5. Referenced documents

{For each document fetched (up to 3) — a Confluence page, an RFC, a design doc, a linked README:}

### {Page title}

**URL:** {full URL}

{Either the full page body if it's short (< 2k chars) or the most relevant excerpt with `[…]` to indicate omissions. Preserve internal headings.}

{If none: `_(no documents referenced)_`}

## 6. Attachments

{Table of every attachment the source reported, plus the download outcome:}

| Filename | Size | Mime | Local path | Status |
|---|---|---|---|---|
| {file1.pdf} | {1.2 MB} | {application/pdf} | `attachments/file1.pdf` | downloaded |
| {file2.png} | {340 KB} | {image/png} | — | skipped (see Context Gaps) |

{If none: `_(no attachments on this ticket)_`}

## 7. Related local docs

{Exhaustive list from Step 5 of the parent skill — every `docs/**/*.md` file that contains the ticket id OR any component name. May be long; do NOT truncate. Format:}

- `{relative-path-from-project-root}` — matched: `{term1}`, `{term2}` — excerpt: `{one line around first match}`

{If none: `_(no related local docs found)_`}

## 8. Context Gaps

{Every failure or absence recorded during Steps 2-5 of the parent skill — attachment download fails, document auth fails, fields this source cannot supply, an empty thread on free-form input, anything the interview should know about. An absence is data: say WHY it is empty so a later reader does not read thinness as laziness. Format:}

- {short description of what was missing or failed, plus what to ask the user about in the interview}

{If clean: `_(none)_`}

---

_(Sections 9+ are appended by the parent skill: § Interview Notes after Step 7, and the master plan is written to a separate file `master-plan.md` in this folder by Step 8.)_
