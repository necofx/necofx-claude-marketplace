# issue.specs — canonical structure

Every `issue.specs` produced by the `create-master-plan` skill MUST use this 9-section structure, in this order, even when a section is empty (use `_(none)_` placeholders).

Sections 10+ (Interview Notes, Plan) are appended later by Steps 7–8 of the parent skill. Do NOT write them in Step 6.

---

# {JIRA-KEY} — {Summary}

**Status:** {status}  
**Issue type:** {issuetype}  
**Priority:** {priority}  
**Reporter:** {reporter displayName} (`{reporter accountId}`)  
**Assignee:** {assignee displayName} (`{assignee accountId}`)  
**Components:** {comma-separated}  
**Labels:** {comma-separated}  
**Fix versions:** {comma-separated}  
**Due date:** {YYYY-MM-DD or `_(none)_`}  
**Parent epic:** {epic key + title, or `_(none)_`}  
**Fetched at:** {YYYY-MM-DD HH:MM}  
**Plan folder:** `{{PLANS_DIR}}/{JIRA-KEY}/`

---

## 1. Description

{Raw Jira description — markdown, verbatim. Do NOT summarise. If empty, write `_(empty in Jira — interview will fill it)_`.}

## 2. Acceptance Criteria

{Verbatim extract of any section in the description whose heading matches `Acceptance Criteria`, `AC`, `Definition of Done`, or `DoD` (case-insensitive). If none, write `_(none in the ticket — to be filled by the interview)_`.}

## 3. Comments

{Chronological. For each comment:}

### {Author display name} · {YYYY-MM-DD HH:MM}

{Comment body, verbatim. Keep ADF / markdown formatting where possible.}

{If no comments: `_(no comments on this ticket)_`}

## 4. Linked Jira issues

{For each linked issue (up to 5, prioritised per SKILL.md Step 4a):}

### {KEY} · {link type} · {status}

**Summary:** {title}

{1-paragraph summary of the linked issue's description — focus on what's relevant to {JIRA-KEY}, not the full text.}

{If none: `_(no linked Jira issues)_`}

## 5. Confluence pages

{For each Confluence page fetched (up to 3):}

### {Page title}

**URL:** {full URL}

{Either the full page body if it's short (< 2k chars) or the most relevant excerpt with `[…]` to indicate omissions. Preserve internal headings.}

{If none: `_(no Confluence pages referenced)_`}

## 6. Attachments

{Table of every attachment metadata returned by Jira, plus the download outcome:}

| Filename | Size | Mime | Local path | Status |
|---|---|---|---|---|
| {file1.pdf} | {1.2 MB} | {application/pdf} | `attachments/file1.pdf` | downloaded |
| {file2.png} | {340 KB} | {image/png} | — | skipped (see Context Gaps) |

{If none: `_(no attachments on this ticket)_`}

## 7. Related local docs

{Exhaustive list from Step 5 of the parent skill — every `docs/**/*.md` file that contains the Jira key OR any component name. May be long; do NOT truncate. Format:}

- `{relative-path-from-project-root}` — matched: `{term1}`, `{term2}` — excerpt: `{one line around first match}`

{If none: `_(no related local docs found)_`}

## 8. Context Gaps

{Every failure recorded during Steps 2–5 of the parent skill — attachment download fails, Confluence auth fails, Jira fields that came back empty when expected, anything the interview should know about. Format:}

- {short description of what was missing or failed, plus what to ask the user about in the interview}

{If clean: `_(none)_`}

---

_(Sections 9+ are appended by the parent skill: § Interview Notes after Step 7, and the master plan is written to a separate file `master-plan.md` in this folder by Step 8.)_
