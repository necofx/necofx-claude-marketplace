# Configuration

The skill in this plugin is its author's own file, shipped byte-for-byte unmodified. That is the point of this package, and it has one consequence you have to deal with before first use: **the skill still carries the three template placeholders that get substituted when it is installed into a repository the original way.**

```
{{JIRA_CLOUD_ID}}    ×5
{{PLANS_DIR}}        ×5
{{TICKET_PREFIX}}    ×6
```

In the original distribution you run `sed` over the copied files and the tokens disappear. Here you cannot: an installed plugin lives in a read-only cache under `~/.claude/plugins/`, and editing it there would be overwritten by the next `/plugin update`.

So the values are supplied from the outside instead — from the repository you are planning in. Paste the block below into that repository's `CLAUDE.md`, which Claude Code loads at the start of every session, so the values are in context before the skill is ever read.

**Be honest with yourself about what this is.** The skill's own text still says `{{JIRA_CLOUD_ID}}`; the block below is what tells the model what that token resolves to. It works because the model reads both. It is weaker than a real substitution, and if a run ever passes a literal `{{...}}` to a tool, this block is the first thing to check.

## The block to copy

```markdown
## Planning workflow configuration

The `create-master-plan` and `decompose-plan` skills carry unresolved `{{TOKEN}}`
placeholders. Resolve them with these values wherever they appear, and never pass a
literal `{{...}}` to a tool:

- `{{PLANS_DIR}}` → `docs/plans`
- `{{TICKET_PREFIX}}` → `GH`
- `{{JIRA_CLOUD_ID}}` → not applicable; this project tracks work in GitHub Issues.

### Ticket source: GitHub, not Jira

Ignore the skill's Atlassian MCP steps (Step 2 fetch, Step 4 linked issues and
Confluence pages) and substitute the `gh` CLI, keeping every downstream step and the
`issue.specs` structure exactly as written:

- Base issue: `gh issue view <N> --json number,title,body,state,labels,assignees,author,comments,milestone`
- Linked work and cross-references: `gh issue view <N> --json body,comments` then follow
  the `#<n>` and PR references it names, honouring the skill's caps of 5 linked issues.
- Referenced PRs: `gh pr view <N> --json title,body,files,state,mergedAt` — the `files`
  list is the highest-value field for grounding the plan in what already landed.
- Attachments: download the `user-images.githubusercontent.com` / `github.com/user-attachments`
  URLs found in the body and comments into `<plan-folder>/attachments/`.

The ticket id is `GH-<issue number>` (e.g. `GH-412`), which satisfies the skill's
`^[A-Z]+-[0-9]+$` validation, and the plan folder is `docs/plans/GH-<number>/`.
```

## Changing the defaults

| Token | Default above | Change it when |
|---|---|---|
| `{{PLANS_DIR}}` | `docs/plans` | Your repo already keeps plans elsewhere. Whatever you choose, `decompose-plan`, `plan-review` and every generated document must be given the same value — they all address the same folder. |
| `{{TICKET_PREFIX}}` | `GH` | You use a real tracker key. Put the key itself here — `ACME`, `PFS` — and ids become `ACME-1234`. |
| `{{JIRA_CLOUD_ID}}` | not applicable | You actually use Jira. Then set it to the bare site host, `your-org.atlassian.net` — no `https://`, no path — drop the GitHub section above, connect the Atlassian MCP (`claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp`), and the skill runs as originally written. |

## If you have no tracker at all

`MANUAL.html` gives the escape hatch, and it costs you nothing downstream: hand-write
`issue.specs` from `skills/create-master-plan/references/issue-specs-template.md` and
start at `/decompose-plan`. Every later step reads that file, not the tracker.
