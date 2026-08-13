# Contributing

A plugin in this marketplace is a folder under `plugins/` plus one row in the root manifest. That is the whole contract, and it is deliberately small: adding a second plugin should never require reorganising the first.

## Layout

```
necofx-claude-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # { name: "necofx", owner, plugins: [ … ] }
├── README.md                     # the shop window: one row per plugin
├── CONTRIBUTING.md
├── LICENSE
└── plugins/
    └── <plugin-name>/
        ├── .claude-plugin/
        │   └── plugin.json       # { name, version, description }
        ├── README.md             # what it does, install, usage, limits
        ├── CLAUDE.md             # rules for an agent working ON this plugin
        ├── skills/<skill>/SKILL.md
        ├── commands/<command>.md
        ├── agents/<agent>.md
        └── …                     # scripts/, rules/, tests/ — all inside the plugin
```

**Everything a plugin needs lives under its own folder**, including `scripts/` and `tests/`. Nothing at the repository root is plugin-specific. A plugin that reaches outside its folder — for a shared script, a shared fixture, a sibling plugin's rules file — breaks the property that makes the next plugin cheap to add.

## Adding a plugin

1. **Create `plugins/<name>/`.** Use the same name in the folder, in `plugin.json`, and in `marketplace.json`. Kebab-case; it is what people type after `/plugin install`.

2. **Write `plugins/<name>/.claude-plugin/plugin.json`:**

   ```json
   {
     "name": "<name>",
     "version": "0.1.0",
     "description": "One sentence, concrete, saying what it produces — not what category it belongs to."
   }
   ```

   Add `mcpServers` only if the plugin actually ships servers.

3. **Add one entry to `.claude-plugin/marketplace.json`:**

   ```json
   { "name": "<name>", "source": "./plugins/<name>", "description": "…" }
   ```

   `source` is a path relative to the repository root. Keep the `description` in step with the one in `plugin.json` — the marketplace one is what shows in `/plugin` listings before anything is installed.

4. **Add one row to the plugin table in the root `README.md`**: name, one line, version, link to the plugin's README. One row, nothing else — the root README is a shop window, and plugin detail belongs in the plugin's own README.

5. **Write `plugins/<name>/README.md`.** A reader arriving from a search engine knows nothing. It has to say what the plugin does, how to install it, what it requires, how to use it end to end, and where it stops being useful. A plugin whose README is a stub is not finished.

## Bumping `version` is the only update signal

`/plugin update <name>` compares the `version` in `plugin.json` against what the installed machine already has. New content in the repo with an unchanged `version` is invisible: users keep running the old copy and nothing tells them otherwise, including you.

So: **any change that alters behaviour gets a version bump in the same commit.** For plugins whose behaviour is defined in prose — skills, commands, agent contracts, templates — "behaviour" includes editing that prose. A reworded instruction in a `SKILL.md` changes what the model does just as surely as a code change would. Typo fixes in a README are the only routine exception.

Semantic versioning, read against the plugin's users: patch for a fix that leaves the interface and the output shape alone; minor for new commands, new skills, or new sections in generated artifacts; major when an existing invocation now does something different, or generated output loses something a downstream consumer relied on.

## Test locally before you push

Point Claude Code at your clone instead of GitHub, and the whole install path is exercised without a round trip:

```
/plugin marketplace add /path/to/your/clone
/plugin install <name>@necofx
```

Verify that the manifests parse, that every skill, command and agent you shipped appears in the listings, and that the commands are invocable from an unrelated directory — plugins get used from repos that are not this one. After editing a plugin file, run `/plugin update <name>` and start a fresh session; a live session may still be holding the version it loaded at start.

## House rules

**English everywhere** — file contents, generated artifacts, commit messages, issue text. This repository is public and its plugins produce documents other people read.

**Self-contained plugins.** No plugin imports conventions, verification commands, rules or examples from any other repository. Whatever a plugin needs, it ships.

**Nothing generated gets committed.** Test sandboxes, scratch output and local overrides belong in `.gitignore`. A plugin's test harness should regenerate its fixtures from scratch on every run rather than depend on committed state.

**Tests never write to public repositories.** If a plugin's tests need a remote — issues, PRs, releases — they create a private, disposable repository and delete it at the end. Synthetic test data seeded into a public repo is visible forever.
