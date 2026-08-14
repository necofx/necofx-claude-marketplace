# necofx-claude-marketplace

Claude Code plugins by necofx. Every plugin lives in its own folder under `plugins/`, carries its own manifest, and documents itself.

## Add the marketplace

The catalogue lives at <https://github.com/necofx/necofx-claude-marketplace>, in `.claude-plugin/marketplace.json`. You register it once per machine; after that, plugins install by name.

From inside a Claude Code session:

```
/plugin marketplace add necofx/necofx-claude-marketplace
```

From a terminal, without a session open — same sources, same effect:

```sh
claude plugin marketplace add necofx/necofx-claude-marketplace
```

**If that fails to clone, you are hitting SSH.** The `owner/repo` shorthand clones over SSH by default, so it needs a GitHub key loaded in `ssh-agent` and `github.com` already in your `known_hosts` — Claude Code suppresses the interactive prompts rather than asking. Give it the HTTPS URL instead:

```
/plugin marketplace add https://github.com/necofx/necofx-claude-marketplace.git
```

or keep the shorthand and export `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1`, which makes every shorthand source clone over HTTPS. Both forms accept a branch or tag if you want to pin: append `@v1` to the shorthand, `#v1` to the URL.

By default the marketplace is declared in your user settings — your machine only. Pass `--scope project` to declare it in the current repo's `.claude/settings.json` instead, so everyone who clones that repo picks it up:

```sh
claude plugin marketplace add necofx/necofx-claude-marketplace --scope project
```

## Install a plugin

```
/plugin install create-master-plan@necofx
```

The `@necofx` suffix is the marketplace `name` declared in `.claude-plugin/marketplace.json`, not the repository name — they only look alike here by coincidence. Running `/plugin` with no arguments opens the browser if you would rather pick from a list.

To confirm what landed:

```sh
claude plugin marketplace list   # is the catalogue registered, and from which source
claude plugin list               # which plugins are installed and enabled
```

## Plugins

| Plugin | What it does | Version | Docs |
|---|---|---|---|
| `create-master-plan` | Step 1: pulls a ticket — **GitHub by default**, Jira/Linear/free-form as adapter profiles — with its links, cited documents and attachments, scans the repo's docs, detects the stack, interviews you over a coverage matrix, and writes `issue.specs` + `master-plan.md`. | 0.3.2 | [README](plugins/create-master-plan/README.md) |
| `decompose-plan` | Step 2: turns that plan into atomic phases grouped into parallel rounds, file-conflict checked and skill-matched, emitting `phases/`, `tasks.md`, `execute-plan.md` and `handoff.md`. | 0.3.3 | [README](plugins/decompose-plan/README.md) |
| `plan-review` | Steps 4–5, optional: generates a self-contained review prompt for a fresh external reviewer — of the plan before it is built, or of the real changeset against the plan afterwards — and offers to run it through the Codex CLI for you. | 0.2.1 | [README](plugins/plan-review/README.md) |

Install instructions specific to a plugin, its tutorial, its limits and its troubleshooting live in that plugin's own README. Nothing about a plugin is duplicated here.

### The three are one workflow

They are separate plugins because they run in separate conversations — that is not packaging convenience, it is the design. Step 1 ends with a large research payload in context; step 3's coordinator needs a near-empty window for the whole plan plus every agent's report. Every step's output is a file, so no step depends on a previous conversation still being open.

```
/create-master-plan 412            →  issue.specs · master-plan.md
        ↓  new conversation
/decompose-plan docs/plans/GH-412  →  phases/ · tasks.md · execute-plan.md · handoff.md
        ↓  new conversation
paste the Coordinator Prompt       →  one agent per phase, a round at a time
        ↓  optional
/plan-implementation-review        →  a prompt for a fresh reviewer, code against plan
```

The last step is a genuine second opinion, not a gate: the workflow closes without it, and `/code-review` inside Claude Code covers the ordinary case.

You can start at step 2: hand-write `issue.specs` and `master-plan.md` and nothing downstream knows the difference. Only step 1 touches a tracker at all, and it reads GitHub by default — Jira, Linear and pasted text are adapter profiles, selected by a detection ladder.

**The workflow is not ours.** It was designed by someone else, who uses it daily and shared their files directly, and it is published here with their permission. The structure, the reasoning and nearly all of the prose are theirs; this repository adds the manifests, the READMEs, and two changes to the skills themselves — the original is hard-wired to Jira and carries install-time placeholders, so the tracker was moved behind an adapter layer with GitHub as the default; and the two review skills gained an optional final step that runs the prompt they generate through the Codex CLI instead of leaving you to run it by hand. Their full manual, updated to match, ships inside each plugin as `MANUAL.html`.

## Updates

Two different things, and mixing them up is the usual reason an update appears not to arrive:

```
/plugin marketplace update necofx   # re-read the catalogue — new plugins, new version numbers
/plugin update <plugin>             # pull the newer plugin itself
```

A plugin's `version` in its `plugin.json` is the only signal an installed machine gets that there is something to pull, and it only sees the bump once the catalogue has been refreshed.

To drop the marketplace entirely, `/plugin marketplace remove necofx` — note this also uninstalls the plugins that came from it.

## Adding a plugin

See [CONTRIBUTING.md](CONTRIBUTING.md). Licensed [MIT](LICENSE).
