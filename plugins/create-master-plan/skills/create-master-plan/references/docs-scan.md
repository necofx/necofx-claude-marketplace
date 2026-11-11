# Step 5 — Scan the local docs/ folder exhaustively

_Moved out of `SKILL.md` so it survives context compaction: the skill reads this file from disk at the step that needs it._

### Step 5 — Scan the local `docs/` folder EXHAUSTIVELY

Per the user's explicit direction, no folder is excluded and no cap is applied beyond glob limits.

1. Build the haystack of search terms:
   - The ticket id itself (e.g. `GH-412`, `ACME-1234`).
   - Every component name from Step 2.
   - The most distinctive nouns from the ticket summary (skip stopwords).
2. Glob `<project-root>/docs/**/*.md` (no exclusion, no depth cap). This glob stays exactly this — nothing here is skipped.
3. For each candidate file, run a case-insensitive Grep for any of the haystack terms. A single hit is enough.
4. Partition matches by where they land, then record each partition differently:
   - **Outside `<plans-root>`, and under `<plans-root>/active/`** — reported in full, exactly as before:
     - relative path from `<project-root>`
     - which terms matched
     - a 1–2 line excerpt around the first match (use Grep with `-B 1 -A 1`)
   - **Under `<plans-root>/closed/`** — collapsed to one line per matching plan folder, no excerpt, however many files inside that folder matched. Do NOT record a path/terms/excerpt per file; instead, once per matching `closed/<ID>/` folder, read that folder's `master-plan.md` header per `references/plan-layout.md`'s "authoritative carrier" rule and record:
     `- <ID> · <status> · closed <date> — matched: <term1>, <term2>`
     If `master-plan.md` is missing, or present but carries no header, use status `unknown` and `closed unknown` for the date field (neither can be read) — never skip the folder.

This produces a potentially long list — that's expected. Section 7 of `issue.specs` is allowed to grow. Collapsing `closed/` matches reduces what the dossier *carries*, not what the scan *reads*: the glob in step 2 still traverses every markdown file under `docs/`, `closed/` included, and step 3 still greps every one of them. Only how much of a `closed/` match gets quoted into `issue.specs` changes — the cost being cut is context, not coverage.
