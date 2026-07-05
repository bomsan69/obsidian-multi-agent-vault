---
command: wiki-research
description: Multi-agent research, saved directly into the vault's 01-Research/ folder
argument-hint: "<topic>" [--breadth=light|standard|exhaustive] [--agents=claude,codex,glm]
---

# /wiki-research — Vault-Aware Multi-Agent Research

1. Read `CLAUDE.md` in the current working directory. Follow its Write Zones and Output Format rules exactly.
2. Parse `$ARGUMENTS` for the topic string plus optional `--breadth` (default: standard) and `--agents` (default: claude,codex,glm) flags.
3. Dispatch the topic to each selected provider in parallel:
   - Claude → overview / architectural framing section (write this yourself)
   - Codex → technical depth section (code, edge cases, benchmarks), via CLI: `codex exec --skip-git-repo-check "…" < /dev/null`
   - GLM → ecosystem section (alternatives, recent developments), via OpenRouter REST API (no CLI needed). Load the key from `.env` in the same shell command:

     KEY=$(grep -E '^OPENROUTER_API_KEY=' .env | cut -d= -f2-)
     curl -s https://openrouter.ai/api/v1/chat/completions \
       -H "Authorization: Bearer $KEY" \
       -H "Content-Type: application/json" \
       -d '{"model": "z-ai/glm-5.2", "messages": [{"role": "user", "content": "…"}]}' \
       | jq -r '.choices[0].message.content'
   - `light` = 1 round per provider, `standard` = 3 rounds, `exhaustive` = full fan-out
   - If a provider is unavailable, proceed with the others and note in the report which ones responded.
   - See `CLAUDE.md` § Secrets Safety — never use `-v`/`--verbose` on curl calls with an `Authorization` header.
4. Score convergence across the returned sections (0-100). This is the `consensus_score` — a
   **Claude estimate** (there is no independent gate): treat it as a perspective-convergence signal,
   not a correctness guarantee. Run `/wiki-review --strict` afterward for actual fact-checking.
5. Build the note body using the required frontmatter:
title: "<topic>"

date: <today, YYYY-MM-DD>

type: research

agents: [<providers actually used>]

consensus_score: <score>

status: draft | reviewed   # "reviewed" only if consensus_score >= 75

followed by one `## 🔵 Claude — ...`, `## 🔴 Codex — ...`, `## 🟣 GLM — ...` section per provider used, and a `## ✅ Consensus Synthesis` section only when `consensus_score >= 75`. If below 75, add a `## ⚠️ Consensus Warning` section listing the specific disagreement points instead.
6. Slugify the topic (lowercase, spaces → hyphens, strip punctuation) to get `<slug>`.
7. Save the note via the Obsidian Local REST API instead of writing the file directly, so Obsidian picks it up immediately. Load the API key from the vault's `.env` in the SAME shell command (shell env does not persist across calls):

KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-)
curl -X PUT http://localhost:27123/vault/01-Research/<slug>.md \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: text/markdown" \
  --data-binary @<tmpfile>

 If the REST API is unreachable (Obsidian closed), fall back to writing the file directly to `01-Research/<slug>.md` and tell the user Obsidian needs to be reopened to see it.
8. Report back a one-line summary: `Consensus: <score>/100 (estimated) — status: <status> — saved to 01-Research/<slug>.md`.

Never write to `05-Published/`. Never skip the frontmatter.