# Ticket source adapters

Step 2 of `SKILL.md` dispatches here instead of fetching from a single tracker. Read this file once at the start of Step 2 and apply the ladder below.

**GitHub is the default.** Everything else is a profile you select into. Nothing downstream cares which one ran: every adapter normalises its fetch into the one intermediate record described below, and Steps 3–9 read that record rather than the tracker. That is the property that makes this extensible — adding a tracker means adding a profile here, never editing `SKILL.md`.

## The normalised record

Every adapter fills this shape. Fields it genuinely cannot supply are left empty and recorded in `issue.specs § Context Gaps` — **an empty field is data, not a defect**, and stating why it is empty stops a later reader assuming the fetch was lazy.

```
id             GH-412 · ACME-1234 · LIN-88 · slug-of-the-title
title
body           markdown, verbatim, never summarised
state
author
assignees      []
labels         []
components     []            may be empty; not every tracker has these
priority                     may be empty
due                          may be empty
parent                       epic / tracking issue, or none
comments       [{author, created, body}]              chronological
links          [{id, relation, state, title, url}]    cap: 5, see below
documents      [{title, url, content}]                cap: 3, see below
attachments    [{filename, mime, size, url}]
source_kind    github | jira | linear | file | free-form
fetched_at     YYYY-MM-DD HH:MM
```

`documents` is the generalisation of "Confluence pages": any cited document worth pulling in whole — a Confluence page, an RFC, a design doc, a linked README. The adapter finds them; the section that holds them does not care where they came from.

## The detection ladder

First match wins. **Ambiguity never guesses — it asks.**

1. **The user said so.** "plan LIN-88 from Linear", "use the Jira one". An explicit instruction beats every probe below.

2. **Argument shape, confirmed by a capability probe.** Shape alone is not enough — `ACME-1234` looks identical to Jira and Linear, so the probe decides.

   | Argument | Probe | Adapter |
   |---|---|---|
   | bare number, `#412`, or a `github.com/*/issues/N` URL | `gh repo view` succeeds | **github** |
   | `^[A-Z]+-[0-9]+$`, or an `*.atlassian.net` URL | Atlassian MCP resolves | **jira** |
   | a `linear.app/*/issue/*` URL, or `^[A-Z]+-[0-9]+$` | Linear MCP resolves | **linear** |
   | a path that exists on disk | — | **file** |
   | anything else | — | **free-form** |

3. **Two adapters match** — say `ACME-1234` with both the Atlassian and Linear MCPs connected. Ask via `AskUserQuestion`, naming both and what each would fetch. Do not pick by ordering; the cost of guessing wrong is a full re-fetch and a dossier pointing at the wrong system.

4. **Nothing matches.** If the argument is absent entirely, ask for it. If it is prose, take the **free-form** adapter rather than failing — a plan from a pasted paragraph is a legitimate input, and Steps 5–9 work identically on it.

Record the chosen adapter and the reason in `issue.specs § Context Gaps` when it was not the obvious one. A reader who finds a thin `links` list deserves to know whether the tracker had no links or the adapter could not read them.

## Caps, and they apply to every adapter

**≤ 5 linked items, ≤ 3 documents.** These keep the fetched context useful without drowning the interview; raise them only when your tickets are unusually sparse. Priority order for links, stopping at 5: parent epic → blocks / is blocked by → relates to → sub-tasks → everything else. Record the relation type alongside each one.

---

## Adapter: GitHub — the default

Requires the `gh` CLI, authenticated (`gh auth status`). No MCP server.

**Base issue.** The id is `GH-<number>`, which satisfies the `^[A-Z]+-[0-9]+$` shape the rest of the skill expects.

```sh
gh issue view <N> --json number,title,body,state,author,assignees,labels,milestone,comments,createdAt,updatedAt,url
```

Map: `title`→title, `body`→body, `state`→state, `labels[].name`→labels, `milestone.title`→parent, `comments[]`→comments. GitHub has no components, priority or due date — leave those empty rather than inventing an equivalent from labels, and say so in Context Gaps if the plan would have used them.

**Links.** GitHub expresses relations three ways, and all three matter:

```sh
# sub-issues and parent, where the repo uses them
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){issue(number:$n){
  parent{number title state} subIssues(first:10){nodes{number title state}}}}}' -F o=<owner> -F r=<repo> -F n=<N>

# cross-references from the timeline: issues and PRs that mention this one
gh issue view <N> --json url && gh api "repos/<owner>/<repo>/issues/<N>/timeline" --jq \
  '.[] | select(.event=="cross-referenced") | {n:.source.issue.number, t:.source.issue.title, pr:(.source.issue.pull_request!=null)}'

# and the plain `#123` references written in the body and comments
```

**Referenced pull requests are the highest-value fetch in this adapter.** For each PR the issue references:

```sh
gh pr view <N> --json title,body,state,mergedAt,files,url
```

The `files` list of a *merged* PR tells you what already landed, which is how "this ticket is gap-closure, not a build" gets discovered cheaply instead of by grepping blind. Carry it into `links[]` and mention it in the interview.

**Documents.** Any non-GitHub URL cited in the body or comments — an RFC, a design doc, a wiki page. Fetch and quote the content into the record; a link is not context.

**Attachments.** GitHub inlines uploads as `github.com/user-attachments/...` or `user-images.githubusercontent.com/...` URLs in the body and comments. Extract them and download into `<plan-folder>/attachments/`. Images are worth reading, not just saving — you can see them.

**Read-only.** No comments, no label edits, no state transitions, no `gh issue edit`. Ever.

---

## Adapter: Jira

Requires the Atlassian MCP: `claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp`. The user must supply their site host (the `your-org.atlassian.net` part of a ticket URL) — ask if it is not in `CLAUDE.md` or the invocation.

The canonical operations are `getJiraIssue`, `getConfluencePage`, `searchConfluenceUsingCql`, `searchJiraIssuesUsingJql` and `fetch`. **Tool-name prefixes vary by install — locate them with `ToolSearch` at runtime and never hardcode a UUID prefix.**

**Base issue.** The id is the Jira key, used verbatim.

```
getJiraIssue
  cloudId: <site host>
  issueIdOrKey: <KEY>
  responseContentFormat: markdown
  fields: ["summary","description","status","labels","components","comment",
           "reporter","assignee","issuetype","parent","priority","duedate",
           "fixVersions","issuelinks","attachment"]
```

This adapter fills `components`, `priority` and `due`, which GitHub cannot.

**Links.** Every `issuelinks` entry plus `parent`, fetched with `fields: ["summary","description","status","issuetype"]`.

**Documents.** Every `https://*.atlassian.net/wiki/...` URL in the description and comments → `getConfluencePage` by page id. If fewer than 3 were found, top up with `searchConfluenceUsingCql` on the summary's distinctive nouns and any component name (`text ~ "<keywords>" AND type = page`, limit 5), keeping the top 3 by title match.

**Attachments.** Use the MCP `fetch` tool if available. If not, write the download command out for the user and continue without blocking:

```sh
curl -L -o "<target-path>" -u "<email>:<api-token>" "<content-url>"
```

**Read-only.** No comments, no transitions, no label edits.

**A failure here never aborts the run.** Auth failures, deleted pages and missing fields go to Context Gaps; the interview covers the hole.

---

## Adapter: Linear

Requires a Linear MCP server. Canonical operations are `get_issue`, `list_comments` and `list_issues` — **resolve the actual tool names with `ToolSearch` (`+linear issue`) rather than assuming a prefix**, exactly as with Atlassian.

**Base issue.** The id is the Linear identifier (`LIN-88`, `ENG-204`), used verbatim.

Fetch the issue by identifier and map: title, description→body, state.name→state, labels, assignee, priority (Linear's numeric priority — carry the label, not the integer), dueDate→due, project or parent→parent. Comments come from `list_comments`.

**Links.** Linear's relations are `blocks` / `blocked by` / `related` / `duplicate`, plus sub-issues and the parent. Same cap of 5 and same priority order.

**Documents.** Linear documents and any external URL cited in the description or comments. Fetch and quote the content.

**Attachments.** Linear attachments carry a URL; download into `<plan-folder>/attachments/` where the URL is reachable, and record the ones that are not.

**Read-only.** No comment creation, no state changes.

---

## Adapter: file, and free-form

For a pasted requirement, a written brief, or a path to a local document. `source_kind` is `file` or `free-form`.

- `id` — `slugify(title)` truncated to 40 characters, **confirmed with the user** before the plan folder is created, because unlike a tracker key nobody else can predict it.
- `title` and `body` — from the text itself. If there is no obvious title, derive one and confirm it in the same question as the id.
- `comments`, `links`, `documents`, `attachments` — empty. State that in Context Gaps as *"free-form input: no tracker thread to pull"*, so the dossier's thinness is explained rather than mistaken for a lazy fetch.

Everything after Step 2 behaves identically. This adapter is the reason the workflow has no hard tracker dependency, and it is the fallback whenever the others cannot authenticate.

---

## Adding a tracker

Add a profile above with the same four headings — base issue, links, documents, attachments — plus its detection row in the ladder. Then stop: **do not edit `SKILL.md`**. If a new tracker needs a change in the skill body, the record above is missing a field, and adding the field is the fix rather than a special case at the call site.
