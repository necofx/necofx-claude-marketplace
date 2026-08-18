# INDEX.md template

The shape `<plans-root>/INDEX.md` takes, per `plan-layout.md` §3.3's "INDEX.md" section. Step 7
creates this file on the first close if it does not already exist, and updates it in place on every
close after that.

```markdown
# Closed plans

| Plan | Title | Status | Closed | PR |
|---|---|---|---|---|
| GH-412 | Partial refunds | completed | 2026-08-18 | #77 |
| GH-388 | Idempotency filter | superseded by GH-412 | 2026-06-02 | #61 |
```

Five columns, in this order:

| Column | Source |
|---|---|
| Plan | `<ID>` |
| Title | The `master-plan.md` H1, with the `<ID> — ` prefix stripped |
| Status | Exactly the `status` field Step 6 wrote into the header |
| Closed | The header's `closed` date |
| PR | The header's `PR` field, without the `PR ` prefix — `#77` or `—` |

List entries newest first — a new row is inserted directly under the header row, never appended at
the bottom. Create the file with just the `# Closed plans` heading and the table header if it does
not exist yet.
