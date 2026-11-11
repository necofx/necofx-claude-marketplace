# Step 7 — Optionally run the review

_Moved out of `SKILL.md` so it survives context compaction: the skill reads this file from disk at the step that needs it._

### Step 7 — Optionally run the review

Running the prompt is **optional and never automatic**. Once it is saved to a file, offer to run it, and run it only if the user agrees:

```bash
(
  if [ -f .codegraph/codegraph.db ]; then   # indexed repo only — see "CodeGraph under read-only" below
    codegraph sync >/dev/null 2>&1 || true   # outside the sandbox: fresh index, oversized WAL healed
    exec 3< <(python3 -c 'import sqlite3, sys, time
c = sqlite3.connect(sys.argv[1]); c.execute("select count(*) from sqlite_master").fetchone()
print("ready", flush=True); time.sleep(86400)' .codegraph/codegraph.db)
    CG_HOLD=$!; trap 'kill $CG_HOLD 2>/dev/null' EXIT
    read -r _ <&3
  fi
  codex exec --sandbox read-only ${CODEX_MODEL:+-m "$CODEX_MODEL"} \
    -o <folder>/codex-implementation-review.md \
    < <folder>/codex-implementation-review-prompt.md
)
```

- Run it **from the repo root**, and as one block: the parentheses scope the CodeGraph holder to this run. The prompt's paths are repo-relative and it regenerates the diff itself.
- **Take the model from the environment, never hardcode one.** If `CODEX_MODEL` is set, pass its value as `-m <name>`; if it is unset, omit `-m` and let the CLI use the model from its own config (Codex's stock default — no special provider or routing). The expansion above does exactly that in one line.
- `--sandbox read-only` is deliberate and sufficient: the reviewer reads the repo and runs read-only git; it must not touch the tree it is judging.
- **CodeGraph under read-only.** The index is SQLite in WAL mode, and opening it means creating `codegraph.db-shm` / `-wal` next to it — which the read-only sandbox forbids, so every `codegraph` call inside Codex fails with `unable to open database file`. It only works when something outside the sandbox already holds the database open: a live CodeGraph daemon usually does in the main checkout, but a worktree's fresh index (the one `create-master-plan` builds) has none, and a daemon idles out after five minutes anyway. Inside the `if`, `codegraph sync` runs first, outside the sandbox: it brings the index up to date with the tree the reviewer is about to read (a worktree index has no file watcher, and a stale index can serve a symbol's code sliced at outdated line numbers), and it lets CodeGraph ≥ 1.6 fold back an oversized write-ahead log on open — a write that would fail inside the sandbox. The rest of the `if` block is the holder: a Python process opens the index, signals `ready` once the sidecar files exist, and is killed when the subshell exits, so it lives exactly as long as the review. The sandbox stays `read-only`. Without `.codegraph/` the block is skipped; without `python3` the `read` returns at once and the reviewer falls back to ordinary search as the Tooling block tells it to.
- `-o <file>` captures the reviewer's final report verbatim, so the user reads the review itself rather than a retelling of it.
- Expect minutes, not seconds, on a large changeset. Run it in the background if the harness supports that, rather than blocking on a foreground timeout.
- **Don't let the tree drift during the run.** The reviewer reproduces the diff live; edits made while it works produce findings against code that no longer exists.
- If `codex` is not installed, or not authenticated (`codex login`), say so and stop. The prompt file is still the deliverable — any capable reviewer with read access to the repo can take it; Codex is only who it was written for.

When the run finishes, point the user at the report file. **Do not summarize it in place of the file**, and do not start applying findings unprompted: they are claims to verify against the code, and accepting a wrong one costs more than missing a marginal one. Invoking a separate reviewer is not the same as reviewing — you still do not review the changeset yourself.
