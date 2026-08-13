# Using a tracker other than GitHub

**You do not need this file to use the plugin.** GitHub is the default and needs no configuration: the skill uses the `gh` CLI, derives the plan id as `GH-<issue number>`, and writes to `docs/plans/GH-<number>/`. If that is your setup, install it and run it.

This file is for the three cases where the default is not what you want.

## The mechanism

Step 2 of the skill reads `skills/create-master-plan/references/source-adapters.md` and applies a detection ladder. Each adapter normalises its fetch into one intermediate record, and **Steps 3–9 read that record rather than the tracker** — which is why switching sources changes one step and nothing else.

The ladder decides by argument shape *confirmed by a capability probe*, because `ACME-1234` looks identical to Jira and Linear. Where two adapters both match, it asks rather than guessing.

---

## Jira

Connect the Atlassian MCP:

```sh
claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp
claude mcp list          # expect: atlassian - Connected
```

First use opens a browser for OAuth. You need read access to the project and to any Confluence spaces your tickets cite.

Then tell the skill your site host, because it cannot discover it — add this to the `CLAUDE.md` of the repository you plan in:

```markdown
## Ticket source

This project tracks work in Jira. Site host: `your-org.atlassian.net`.
Plan ids are the Jira key verbatim (e.g. `ACME-1234`).
```

The host is the bare `your-org.atlassian.net` part of a ticket URL — no `https://`, no path. With the MCP connected, `/create-master-plan ACME-1234` takes the Jira adapter, which additionally fills `components`, `priority` and `due` that GitHub has no equivalent for, and pulls Confluence pages into `## 5. Referenced documents`.

## Linear

Connect a Linear MCP server, then:

```markdown
## Ticket source

This project tracks work in Linear. Plan ids are the Linear identifier
verbatim (e.g. `ENG-204`).
```

The skill resolves the actual MCP tool names at runtime rather than assuming a prefix, so any Linear MCP that exposes the canonical `get_issue` / `list_comments` operations works.

**If both the Atlassian and Linear MCPs are connected**, a bare `ENG-204` matches both adapters and the skill will ask which one you meant. The `CLAUDE.md` line above is what stops it having to ask every time.

## A different plans directory

The default is `docs/plans/<TICKET-ID>/`. To change it:

```markdown
## Planning workflow

Plan folders live under `docs/prps/<TICKET-ID>/`, one per ticket.
```

Whatever you choose, `decompose-plan` and `plan-review` must be given the same value — all three address the same folder, and a mismatch means the second step cannot find what the first one wrote.

## No tracker at all

Nothing to configure. Paste the requirement, or point the skill at a local document:

```
/create-master-plan
```

It takes the free-form adapter, asks you to confirm a slug for the plan id, and leaves `comments`, `links` and `documents` empty — recorded in `## 8. Context Gaps` as *"free-form input: no tracker thread to pull"*, so the dossier's thinness is explained rather than mistaken for a lazy fetch. Everything from the `docs/` scan onward behaves identically.

You can also skip this step entirely: hand-write `issue.specs` from `skills/create-master-plan/references/issue-specs-template.md` and start at `/decompose-plan`. Steps 2–5 of the workflow have no tracker dependency.

---

## Adding a tracker that is not listed

Add a profile to `skills/create-master-plan/references/source-adapters.md` with the same four headings the others use — base issue, links, documents, attachments — plus its row in the detection ladder. **Do not edit `SKILL.md`.** If a new tracker seems to need a change in the skill body, the normalised record is missing a field, and adding the field is the fix rather than a special case at the call site.
