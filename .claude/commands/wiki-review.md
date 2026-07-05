---
command: wiki-review
description: Structured review of an existing vault note, saved into 03-Reviews/
argument-hint: "<vault-path>" [--strict]
---

# /wiki-review — Vault-Aware Multi-Agent Review

1. Read `CLAUDE.md` and follow its Write Zones / Output Format rules.
2. Parse `$ARGUMENTS` for the note path (e.g. `01-Research/semantic-caching-llms.md`) and the optional `--strict` flag.
3. Fetch the note's current content. Load the key from `.env` in the same shell command: `KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-); curl -H "Authorization: Bearer $KEY" http://localhost:27123/vault/<path>`. Reuse `$KEY` for the PUT saves in steps 7-8.
4. Run a structured review with Claude and Codex (GLM optional):
   - Check technical accuracy, completeness, and internal consistency of each section.
   - If `--strict` is set, flag every factual claim that cannot be independently confirmed with `[VERIFY]` in a dedicated "Action Items" section — do not silently accept unverifiable claims.
5. Compute a `review_score` (0-100) reflecting the reviewers' confidence in the note as-is.
6. Write the review note with frontmatter `type: review`, referencing the source note path, plus the review body (per-reviewer sections, an "Action Items" list, and a summary).
7. Save the review via REST API PUT to `03-Reviews/<slug>-review.md`.
8. Update the *original* note's frontmatter (not its body) by fetching it, adding/updating `reviewed_at: <today>` and `review_score: <score>`, and PUTting it back — do not touch any other content in the original note.
9. Report: `Review score: <score>/100 — [VERIFY] items: <count> — saved to 03-Reviews/<slug>-review.md`.

Never write to `05-Published/`. Never overwrite the source note's body — only its frontmatter.