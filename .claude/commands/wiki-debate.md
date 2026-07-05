---
command: wiki-debate
description: Structured multi-agent debate, saved into the vault's 02-Debates/ folder
argument-hint: "<topic>" [--rounds=N]
---

# /wiki-debate — Vault-Aware Multi-Agent Debate

1. Read `CLAUDE.md` and follow its Write Zones / Output Format rules.
2. Parse `$ARGUMENTS` for the topic and optional `--rounds` (default: 2: opening + rebuttal).
3. Check whether a matching note already exists in `01-Research/` (same slug or close topic match). Load the key from `.env` in the same shell command: `KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-); curl -H "Authorization: Bearer $KEY" http://localhost:27123/vault/01-Research/`. If found, read it and use it as shared context for all providers, and note the relationship for step 6.
4. Run the debate:
   - Each active provider (Claude, Codex, Gemini) states an explicit position (not just a description).
   - For each additional round, providers respond directly to the other providers' previous points (rebuttals) — this is where blind spots should surface.
   - After the final round, attempt synthesis. Only write a `## ✅ Consensus Synthesis` if the positions genuinely converged; otherwise list each open disagreement under `## 🚧 Unresolved Points`.
5. Build the note with the same required frontmatter as research notes, but `type: debate`.
6. If a related research note was found in step 3, add a `[[wikilink]]`-style backlink to it in the new debate note, and append a backlink to the debate note at the bottom of the original research note (via the REST API POST/append endpoint) so the link is bidirectional.
7. Save the note via REST API PUT to `02-Debates/<slug>.md` (same fallback-to-direct-write rule as `/wiki-research` if the API is unreachable).
8. Report: `Consensus: <score>/100 — status: <status> — saved to 02-Debates/<slug>.md`.

Never write to `05-Published/`.

