# Your Obsidian Brain — Multi-Agent Vault (Corrected Guide)

> Corrected edition. Fixes the Local REST API **authentication/port** changes (v4.x now
> requires an API key), the folder-structure diagram, and several outdated claims. The
> public git repo referenced in the original article no longer exists, so this guide is
> **self-contained**: every file you need is inline below — no `git clone` required.
>
> **Note:** the original article routed everything through the *Claude Octopus* plugin. This
> vault instead calls the Codex CLI directly and GLM via OpenRouter's REST API from its own
> `/wiki-*` commands — no plugin dependency, no Gemini CLI. The consensus score is a Claude
> estimate, and `/wiki-review --strict` is the real fact-check.

No language model admits its own blind spots. Claude is excellent at synthesis and
architecture, but can be overly cautious on technical claims. Codex is surgical at code
analysis and edge cases, but loses system context. GLM has broad ecosystem awareness,
but its self-reported specifics (benchmarks, pricing, token counts) need independent
verification more often than the other two.

The solution isn't to pick one of the three. It's to put all three on the same problem,
record what each one says, and apply a consensus gate before trusting the result.

## Table of Contents

1. What Obsidian Is and Why It's Free
2. Installing Obsidian and Creating Your First Vault
3. The Local REST API Plugin — The Bridge Between Agents and Vault
4. Installing Claude Code
5. Adding Codex and GLM to the Mix
6. The Multi-Agent Vault Structure
7. The CLAUDE.md: The Contract All Agents Obey
8. The Four Slash Commands
9. How the Consensus Gate Works
10. Complete Walkthrough
11. Persistence Across Sessions
12. Honest Limitations

---

## 1. What Obsidian Is and Why It's Free

Obsidian is a Markdown-based note editor that runs entirely locally — no server, no
account, no required internet connection. Your notes are plain `.md` files in a folder on
your computer. Obsidian is just a visualization layer on top of files you already own.

**Why this matters:** AI agents write and read text files. If the vault is a folder of
`.md` files, agents can interact with it via the filesystem or via a local HTTP API — no
proprietary lock-in.

**Pricing:** Obsidian is free for personal use. Optional paid products (Sync ~$4/mo,
Publish ~$8/mo) are **not** needed for this setup. All local functionality — including the
community REST API plugin — is free.

## 2. Installing Obsidian and Creating Your First Vault

### 2.1 Download

Go to [obsidian.md/download](https://obsidian.md/download) and grab the installer for your
OS (macOS `.dmg`, Windows `.exe`, Linux `.AppImage`/`.deb`). No account required.

### 2.2 Create the vault

On first launch choose **"Create new vault"**:

- **Vault name:** `OctopusVault`
- **Location:** any folder — this guide uses `~/Documents/Vaults/OctopusVault`

> Use one consistent path everywhere. Wherever you put it, that folder **is** the vault.

### 2.3 Understanding what a vault is

An Obsidian vault is literally a folder. Any script or AI agent that reads/writes files can
interact with it:

```bash
# The vault is just a folder of .md files
ls ~/Documents/Vaults/OctopusVault/
# .obsidian/   CLAUDE.md   00-Inbox/   01-Research/   ...
```

## 3. The Local REST API Plugin — The Bridge Between Agents and Vault

The **Local REST API** plugin (by Adam Coddington / `coddingtonbear`) exposes a local HTTP
server that lets you read and write vault notes via `curl`. It's what lets Claude Code write
a note that appears in Obsidian immediately.

> ⚠️ **Changed in v4.x:** the plugin is now "Local REST API with MCP" and **requires an API
> key (Bearer token) on every `/vault/**` request**. The old "no authentication by default"
> behavior is gone. All examples below include the key.

### 3.1 Why the REST API instead of writing files directly?

You *can* just `echo "content" > .../note.md`. The difference:

- Obsidian doesn't always detect external file changes in real time.
- The REST API notifies Obsidian immediately — the note appears without reloading.
- It handles encoding, conflicts, and hot-reload correctly.

### 3.2 Installing the plugin and getting your API key

1. **Settings** (gear, bottom-left) → **Community plugins**
2. If "Safe mode" is on, click **Turn on community plugins** (one-time)
3. **Browse** → search `Local REST API` → the result by **coddingtonbear** → **Install** → **Enable**
4. Open the plugin's settings. **Copy the API Key** shown at the top — you need it for every
   request. This step did not exist in older versions and is easy to miss.

Default ports:

- **HTTPS `27124`** — the secure default (self-signed cert → `curl -k`)
- **HTTP `27123`** — plain HTTP, optional/opt-in

This guide uses HTTP `27123` for readability; switch to `https://localhost:27124` with
`-k` if you prefer the encrypted endpoint.

### 3.3 Verifying it works

```bash
# Put your key in an env var so it never appears inline
export OBSIDIAN_API_KEY="YOUR_API_KEY"   # from plugin settings > API Key

curl http://localhost:27123/ -H "Authorization: Bearer $OBSIDIAN_API_KEY"
```

Expected (note the real fields — version is 4.x, and `authenticated` reflects your key):

```json
{"status":"OK","service":"Obsidian Local REST API",
 "versions":{"obsidian":"1.12.7","self":"4.1.3"},
 "authenticated":true}
```

If you call `/` **without** the key you'll still get `"status":"OK"` but
`"authenticated":false` — that only proves the server is up, **not** that writes will work.
To truly verify, list the vault (this requires the key):

```bash
curl http://localhost:27123/vault/ -H "Authorization: Bearer $OBSIDIAN_API_KEY"
```

If this returns `401`, your key is wrong or missing.

### 3.4 Main endpoints the system uses

```bash
# List vault files
curl http://localhost:27123/vault/ \
  -H "Authorization: Bearer $OBSIDIAN_API_KEY"

# Create or overwrite a note
curl -X PUT http://localhost:27123/vault/01-Research/my-note.md \
  -H "Authorization: Bearer $OBSIDIAN_API_KEY" \
  -H "Content-Type: text/markdown" \
  --data-binary @local-file.md

# Read an existing note
curl http://localhost:27123/vault/01-Research/my-note.md \
  -H "Authorization: Bearer $OBSIDIAN_API_KEY"

# Append to an existing note (used by /wiki-debate for backlinks)
curl -X POST http://localhost:27123/vault/01-Research/my-note.md \
  -H "Authorization: Bearer $OBSIDIAN_API_KEY" \
  -H "Content-Type: text/markdown" \
  --data-binary $'\n\n## Added section\nNew content'
```

The server binds to localhost, but **the API key is still required** — do not commit it to a
public repo (see §12).

## 4. Installing Claude Code

### 4.1 Claude Code

```bash
# Requires Node.js 18+
npm install -g @anthropic-ai/claude-code
claude --version
```

First run in a folder triggers browser-based sign-in with your Anthropic account.

> **No orchestration plugin required.** This vault does its multi-model fan-out by calling the
> Codex CLI and GLM (via OpenRouter's REST API) directly from the `/wiki-*` commands (§8) — there
> is no external plugin dependency and no Gemini CLI. Claude alone works too; extra providers
> simply add more independent perspectives.

## 5. Adding Codex and GLM to the Mix

Both are optional. Codex authenticates via browser OAuth; GLM needs only an OpenRouter API key
(no CLI to install).

### 5.1 Codex (OpenAI)

```bash
npm install -g @openai/codex
codex login      # opens browser, uses your OpenAI account
codex --version
```

> The exact underlying model and whether it's included with a given ChatGPT plan varies over
> time — check OpenAI's current terms rather than assuming a fixed model name.

### 5.2 GLM (via OpenRouter)

No CLI install required — GLM is called as a plain REST API.

1. Get an API key at [openrouter.ai/keys](https://openrouter.ai/keys).
2. Add it to `.env`: `OPENROUTER_API_KEY=...` (see `.env.example`).
3. Confirm the exact current model slug at [openrouter.ai/models](https://openrouter.ai/models)
   before relying on it — OpenRouter's model IDs change as providers ship new versions (this
   guide uses `z-ai/glm-5.2` as of writing).

> Why not Gemini CLI: its free "Gemini Code Assist for individuals" OAuth tier can be blocked
> outright (`IneligibleTierError`) with no CLI-side fix. OpenRouter sidesteps this — one API
> key, one REST endpoint, nothing to re-authenticate.

### 5.3 Verifying providers

```bash
codex --version   # Codex reachable?

# GLM reachable? (needs OPENROUTER_API_KEY in .env)
KEY=$(grep -E '^OPENROUTER_API_KEY=' .env | cut -d= -f2-)
curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "z-ai/glm-5.2", "messages": [{"role": "user", "content": "reply OK"}]}' \
  | jq -r '.choices[0].message.content'
```

> See `CLAUDE.md` § Secrets Safety — never add `-v`/`--verbose` to this call, it would print
> the API key in plaintext.

With Claude + Codex + GLM reachable you get three independent perspectives. The `/wiki-*`
commands dispatch to whichever providers are available and note in the report which ones responded.

## 6. The Multi-Agent Vault Structure

Since the original template repo is gone, build the structure by hand:

```bash
cd ~/Documents/Vaults/OctopusVault
mkdir -p 00-Inbox 01-Research 02-Debates 03-Reviews 04_korean 05-Published .claude/commands
```

Resulting layout:

```text
OctopusVault/
├── CLAUDE.md            # Agent contract — all models read this first
├── 00-Inbox/            # Raw captures (human only)
├── 01-Research/         # /wiki-research output (all agents)
├── 02-Debates/          # /wiki-debate output (all agents)
├── 03-Reviews/          # /wiki-review output (all agents)   ← note the plural "s"
├── 04_korean/          # /translate output — Korean translations (all agents)
├── 05-Published/        # Final, curated notes (human only)
└── .claude/
    └── commands/        # Slash command definitions (vault root, NOT under 05-Published)
        ├── wiki-research.md
        ├── wiki-debate.md
        ├── wiki-review.md
        └── translate.md
```

> **Two corrections vs. the original diagram:** (1) the folder is `03-Reviews/` (plural) —
> the code and CLAUDE.md all use the plural, so the folder must match. (2) `.claude/` lives
> at the **vault root**, *not* inside `05-Published/`. Putting command definitions under
> `05-Published/` would contradict the "agents never write to 05-Published" rule.

**The golden rule:** agents write to `01-Research/`, `02-Debates/`, `03-Reviews/`, and
`04_korean/`. Only you move notes to `05-Published/`.

## 7. The CLAUDE.md: The Contract All Agents Obey

Claude Code auto-reads any `CLAUDE.md` in the working directory. Save this as
`~/Documents/Vaults/OctopusVault/CLAUDE.md`:

```markdown
## Vault Identity

This vault is a multi-agent knowledge base for technical research.
Primary language: Korean          # set this to your own primary language
Tone: Technical, precise, citation-conscious
Audience: Software engineers and AI practitioners

## Write Zones

| Zone      | Path           | Who writes  |
|-----------|----------------|-------------|
| Inbox     | 00-Inbox/      | Human only  |
| Research  | 01-Research/   | All agents  |
| Debates   | 02-Debates/    | All agents  |
| Reviews   | 03-Reviews/    | All agents  |
| Published | 05-Published/  | Human only  |
| Korean    | 04_korean/     | All agents  |

Rule: Agents must never write to 05-Published/. The 04_korean/ zone holds /translate output and is agent-writable.

## Output Format

Every agent-written note must start with this frontmatter:

---
title: ""
date: YYYY-MM-DD
type: research | debate | review
agents: [claude, codex, glm]
consensus_score: 0-100
status: draft | reviewed | published
---

Each agent signs its own section in the body:

## 🔵 Claude — [section name]
## 🔴 Codex — [section name]
## 🟣 GLM — [section name]
## ✅ Consensus Synthesis   (only if consensus_score >= 75)

## Session Start

At the start of each session:
1. Read 00-Inbox/ for pending items
2. Read the last 3 notes in 01-Research/ for context continuity
3. Check 02-Debates/ for unresolved debates (status: draft)
```

> Keep `Primary language` matching what you actually want. (The original article's example
> said English while the working vault used Korean — pick one and be consistent.)

## 8. The Four Slash Commands

Create these four files under `.claude/commands/`. **Replace `YOUR_API_KEY`** with the key
from §3.2 — and never push that key to a public repo (§12).

### 8.1 `.claude/commands/wiki-research.md`

```markdown
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
   - GLM → ecosystem section (alternatives, recent developments), via OpenRouter REST API (no CLI needed):

     KEY=$(grep -E '^OPENROUTER_API_KEY=' .env | cut -d= -f2-)
     curl -s https://openrouter.ai/api/v1/chat/completions \
       -H "Authorization: Bearer $KEY" \
       -H "Content-Type: application/json" \
       -d '{"model": "z-ai/glm-5.2", "messages": [{"role": "user", "content": "…"}]}' \
       | jq -r '.choices[0].message.content'
   - `light` = 1 round per provider, `standard` = 3 rounds, `exhaustive` = full fan-out
   - See `CLAUDE.md` § Secrets Safety — never use `-v`/`--verbose` on curl calls with an `Authorization` header.
4. Score convergence across the returned sections (0-100). This is the `consensus_score` — a
   Claude estimate, not an independent gate. Run `/wiki-review --strict` afterward for the real fact-check.
5. Build the note body using the required frontmatter (title, date, type: research, agents, consensus_score, status).
   status is "reviewed" only if consensus_score >= 75, otherwise "draft".
   Include one `## 🔵 Claude — ...`, `## 🔴 Codex — ...`, `## 🟣 GLM — ...` section per provider used.
   Add `## ✅ Consensus Synthesis` only when consensus_score >= 75; otherwise a `## ⚠️ Consensus Warning` listing disagreements.
6. Slugify the topic (lowercase, spaces → hyphens, strip punctuation) to get `<slug>`.
7. Save the note via the Obsidian Local REST API so Obsidian picks it up immediately:

   KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-)
   curl -X PUT http://localhost:27123/vault/01-Research/<slug>.md \
     -H "Authorization: Bearer $KEY" \
     -H "Content-Type: text/markdown" \
     --data-binary @<tmpfile>

   If the REST API is unreachable (Obsidian closed), fall back to writing the file directly to
   `01-Research/<slug>.md` and tell the user to reopen Obsidian.
8. Report: `Consensus: <score>/100 (estimated) — status: <status> — saved to 01-Research/<slug>.md`.

Never write to `05-Published/`. Never skip the frontmatter.
```

### 8.2 `.claude/commands/wiki-debate.md`

```markdown
---
command: wiki-debate
description: Structured multi-agent debate, saved into the vault's 02-Debates/ folder
argument-hint: "<topic>" [--rounds=N]
---

# /wiki-debate — Vault-Aware Multi-Agent Debate

1. Read `CLAUDE.md` and follow its Write Zones / Output Format rules.
2. Parse `$ARGUMENTS` for the topic and optional `--rounds` (default: 2 = opening + rebuttal).
3. Check whether a matching note exists in `01-Research/`. Load the key from `.env` in the same
   shell command:
   KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-)
   curl -H "Authorization: Bearer $KEY" http://localhost:27123/vault/01-Research/
   If found, read it and use it as shared context for all providers.
4. Run the debate: each active provider (Claude, Codex, GLM) states an explicit position; each
   extra round adds rebuttals. After the final round attempt synthesis. Write
   `## ✅ Consensus Synthesis` only on genuine convergence; otherwise list open items under
   `## 🚧 Unresolved Points`.
5. Build the note with the same required frontmatter, but `type: debate`.
6. If a related research note was found, add a `[[wikilink]]` backlink in the debate note and
   append a backlink to the debate at the bottom of the research note via the REST API POST endpoint.
7. Save via REST API PUT to `02-Debates/<slug>.md` (same fallback rule as /wiki-research):
   curl -X PUT http://localhost:27123/vault/02-Debates/<slug>.md \
     -H "Authorization: Bearer $KEY" \
     -H "Content-Type: text/markdown" --data-binary @<tmpfile>
8. Report: `Consensus: <score>/100 — status: <status> — saved to 02-Debates/<slug>.md`.

Never write to `05-Published/`.
```

### 8.3 `.claude/commands/wiki-review.md`

```markdown
---
command: wiki-review
description: Structured review of an existing vault note, saved into 03-Reviews/
argument-hint: "<vault-path>" [--strict]
---

# /wiki-review — Vault-Aware Multi-Agent Review

1. Read `CLAUDE.md` and follow its Write Zones / Output Format rules.
2. Parse `$ARGUMENTS` for the note path (e.g. `01-Research/semantic-caching-llms.md`) and optional `--strict`.
3. Fetch the note's content. Load the key from `.env` in the same shell command:
   KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-)
   curl -H "Authorization: Bearer $KEY" http://localhost:27123/vault/<path>
4. Run a structured review with Claude and Codex (GLM optional): check technical accuracy,
   completeness, internal consistency. If `--strict`, flag every unverifiable factual claim with
   `[VERIFY]` in a dedicated "Action Items" section.
5. Compute a `review_score` (0-100).
6. Write the review note with `type: review`, referencing the source path, plus per-reviewer
   sections, an "Action Items" list, and a summary.
7. Save via REST API PUT to `03-Reviews/<slug>-review.md`.
8. Update the *original* note's frontmatter only (fetch, add/update `reviewed_at` and
   `review_score`, PUT back) — do not touch the original body.
9. Report: `Review score: <score>/100 — [VERIFY] items: <count> — saved to 03-Reviews/<slug>-review.md`.

Never write to `05-Published/`. Never overwrite the source note's body — only its frontmatter.
```

### 8.4 `.claude/commands/translate.md`

```markdown
---
command: translate
description: Translate an English tech note into natural Korean, saved into 04_korean/
argument-hint: "<vault-path>" | "<raw English text>"
---

# /translate — English → Korean Technical Translation

1. Read `CLAUDE.md` and follow its Write Zones rules. Never write to `05-Published/`.
2. Parse `$ARGUMENTS`. If it looks like a vault path (e.g. `01-Research/foo.md`), fetch it:
   curl http://localhost:27123/vault/<path> -H "Authorization: Bearer YOUR_API_KEY"
   Otherwise treat `$ARGUMENTS` as raw English text to translate directly.
3. Translate meaning-first (의역 우선), not literally. Keep industry-standard English terms
   (LLM, RAG, PEFT, VRAM, Docker, JSON, fine-tuning, …); render everything else as idiomatic Korean.
4. Preserve the Markdown structure exactly (headings, tables, code blocks, frontmatter). Do NOT
   translate code inside code blocks; DO translate code comments to Korean. You may translate the
   frontmatter `title` value but leave keys/enums intact. Add no content the source lacks
   (use `> **역주**: ...` only if a translator's note is truly needed).
5. Save to `04_korean/<same-filename>` via REST API PUT:
   curl -X PUT http://localhost:27123/vault/04_korean/<basename> \
     -H "Authorization: Bearer YOUR_API_KEY" \
     -H "Content-Type: text/markdown" --data-binary @<tmpfile>
6. Report: `Translated <basename> → 04_korean/<basename>`.

Never write to `05-Published/`.
```

> The real command file embeds a fuller term-preservation policy (keep-in-English vs
> translate-to-Korean tables, tone rules, tricky-construct mappings). See
> `.claude/commands/translate.md` in the repo for the complete version.

### Usage

```bash
/wiki-research "semantic caching for LLMs"
/wiki-research "LoRA vs QLoRA fine-tuning tradeoffs" --breadth=exhaustive
/wiki-debate "monorepo vs microservices for my project" --rounds=3
/wiki-review 01-Research/semantic-caching-llms.md --strict
/translate 01-Research/lora-vs-qlora-fine-tuning-tradeoffs.md
```

## 9. How the Consensus Gate Works

The `consensus_score` is a **perspective-convergence signal**: after the provider sections come
back, Claude reads them and estimates (0-100) how strongly they agree. A note scores 75+ when the
sections point the same direction. This is an *estimate by Claude*, not an independent algorithm —
and, as §12 notes, agreement is not the same as correctness. `/wiki-review --strict` is the real
fact-check.

A note scoring **< 75** isn't a failure — it's a diagnosis:

```markdown
## ⚠️ Consensus Warning
consensus_score: 52
status: draft
Disagreement points requiring human review:
- Claude and Codex disagree on optimal cache threshold (0.88 vs 0.95)
- GLM suggests managed services; Claude and Codex prefer self-hosted
```

## 10. Complete Walkthrough

**Before starting:** Obsidian open with `OctopusVault`, REST API plugin enabled, key exported.

```bash
export OBSIDIAN_API_KEY="YOUR_API_KEY"
curl -s http://localhost:27123/vault/ -H "Authorization: Bearer $OBSIDIAN_API_KEY"
```

If that lists your files (not `401`), you're ready.

```bash
cd ~/Documents/Vaults/OctopusVault
claude
/wiki-research "semantic caching for LLMs"
```

Expected terminal flow:

```text
Dispatching to available providers…
  🔵 Claude   → overview…
  🔴 Codex    → technical depth…
  🟣 GLM      → ecosystem…
⚖️  Consensus (estimated): 82/100
💾 Saving to vault: 01-Research/semantic-caching-llms.md
✅ Done — status: reviewed
```

The note appears in Obsidian automatically. Frontmatter includes the `type` field:

```yaml
---
title: "Semantic Caching for LLMs"
date: 2026-06-16
type: research
agents: [claude, codex, glm]
consensus_score: 82
status: reviewed
---
```

Then deepen and review:

```bash
/wiki-debate "semantic cache vs no cache for LLM APIs"
/wiki-review 01-Research/semantic-caching-llms.md --strict
```

## 11. Persistence Across Sessions

Each new session, Claude Code re-reads `CLAUDE.md`, which instructs it to read `00-Inbox/`,
the last 3 notes in `01-Research/`, and open debates in `02-Debates/`. The vault is the
memory — **semantic** memory (what's written) rather than episodic (what happened in one
chat). Use Obsidian's graph view (Cmd/Ctrl+G) to see how notes connect.

## 12. Honest Limitations

- **Token cost.** Three models in parallel roughly triples consumption. For routine work,
  `/wiki-research --agents=claude` is fine; use all three when the decision justifies it.
- **The gate doesn't guarantee correctness.** 82/100 means three models *agree*, not that
  they're *right*. Items marked `[VERIFY]` still need primary-source checking.
- **Obsidian must be open** for the REST API. If closed, the commands fall back to writing
  files directly; notes appear once you reopen Obsidian.
- **Keep your API key out of version control.** The key is required on every request, and the
  `.claude/commands/*.md` files embed it literally. If you publish this vault:
  1. **Rotate** the key in the plugin settings, and
  2. add `.claude/` (or at least the key) to `.gitignore`.
  A leaked key grants read/write to your entire vault.
- **Vault decay.** Notes accumulate faster than you curate. Periodic archiving is a human job.
```