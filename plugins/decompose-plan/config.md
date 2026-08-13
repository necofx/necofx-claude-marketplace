# Configuration

The skill in this plugin is its author's own file, shipped byte-for-byte unmodified. It carries
one unresolved template placeholder, substituted with `sed` when the workflow is installed into
a repository the original way, and which you cannot edit here — an installed plugin lives in a read-only cache
under `~/.claude/plugins/`, and any edit there is discarded by the next `/plugin update`.

```
{{TICKET_PREFIX}}    ×3
```

It appears only in the generated phase headings (`{PLAN_KEY} / Phase NN — Title`) and in the
skill's trigger examples, so the blast radius is small: a wrong value produces oddly-named
phase headings, not a failed run. It is still worth setting.

Supply it from the repository you are planning in — paste this into that repo's `CLAUDE.md`,
which Claude Code loads at the start of every session:

```markdown
## Planning workflow configuration

The `decompose-plan` skill carries an unresolved `{{TICKET_PREFIX}}` placeholder. Resolve it
to `GH` (this project tracks work in GitHub Issues), and never pass a literal `{{...}}` to a
tool or into a generated file. Plan folders live under `docs/plans/GH-<issue number>/`.
```

If you are also running [`create-master-plan`](../create-master-plan/config.md), you already
have a block covering this token — use that one and do not write a second. Two blocks
declaring the same token is how they end up disagreeing.

Replace `GH` with your real tracker key (`ACME`, `PFS`) if you have one; ids become
`ACME-1234` and the folder `docs/plans/ACME-1234/`. The value must match whatever
`create-master-plan` used, because both address the same folder.

## Starting here, without step 1

This skill needs a folder containing a master plan and nothing else — it never touches a
tracker. If you have no Jira and do not want the GitHub adaptation, hand-write `issue.specs`
and `master-plan.md` and point `/decompose-plan` at the folder. `MANUAL.html` is explicit
that steps 2–5 have no tracker dependency.
