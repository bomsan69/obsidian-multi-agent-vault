# Obsidian Multi-Agent Vault

A local **Obsidian knowledge vault** where **Claude, Codex, and Gemini** research, debate,
and review notes — then a human curates what gets published. No orchestration plugin, no
lock-in: just plain `.md` files, the Obsidian Local REST API, and four slash commands.

> **Why:** No single LLM admits its own blind spots. Put the same problem to several models,
> **record what each one says**, and **verify before you trust it** — the value comes from the
> `[VERIFY]`/`[FIX]` items a review surfaces, not from a "consensus" number.

---

## What's inside

| Component | Role |
|---|---|
| **Obsidian Vault** (`.md` files) | Persistent memory — semantic knowledge that survives across sessions |
| **`/wiki-*` commands** → Codex/Gemini CLIs (direct) | Multi-model orchestration, **plugin-free** |
| **`/wiki-review --strict`** | Verification — flags unconfirmable claims as `[VERIFY]` |
| **`CLAUDE.md`** | The contract every agent obeys (write zones, output format, session rules) |

### The four commands (`.claude/commands/`)

- **`/wiki-research <topic>`** — parallel research across models → `01-Research/`
- **`/wiki-debate <topic>`** — structured positions + rebuttals → `02-Debates/`
- **`/wiki-review <path> [--strict]`** — Claude+Codex review → `03-Reviews/`, writes `review_score` back
- **`/translate <path>`** — English → natural Korean, terms preserved → `04_korean/`

### Vault zones

```text
00-Inbox/     Human only — raw captures
01-Research/  Agents — /wiki-research output
02-Debates/   Agents — /wiki-debate output
03-Reviews/   Agents — /wiki-review output
04_korean/    Agents — /translate output
05-Published/ Human only — curated, final notes
```

**Golden rule:** agents write to `01`–`03` and `04_korean`; only a human moves notes to
`05-Published/`. That separation is what prevents unverified content from being "published".

---

## Requirements

- [Obsidian](https://obsidian.md) + the **Local REST API** plugin (coddingtonbear, **v4.x**)
- [Claude Code](https://claude.com/claude-code) (`npm install -g @anthropic-ai/claude-code`)
- *(optional, for extra perspectives)* [Codex CLI](https://github.com/openai/codex) and
  [Gemini CLI](https://github.com/google-gemini/gemini-cli)

> The Local REST API plugin now **requires an API key (Bearer token)** on every `/vault/**`
> request — the old "no auth" behavior is gone. See [`GUIDE.md`](GUIDE.md) §3.

## Quick start

```bash
git clone https://github.com/bomsan69/obsidian-multi-agent-vault.git
cd obsidian-multi-agent-vault

# 1) Open the folder as a vault in Obsidian, enable the Local REST API plugin, copy its API Key.
# 2) Configure secrets (never committed):
cp .env.example .env       # then paste OBSIDIAN_API_KEY (and GEMINI_API_KEY if using Gemini)

# 3) Start Claude Code inside the vault:
claude
/wiki-research "your topic here"
```

Full walkthrough (REST API auth, provider setup, structure, commands): **[`GUIDE.md`](GUIDE.md)**.
Background & design rationale: **[`blog-verified-second-brain.md`](blog-verified-second-brain.md)**.

---

## Honest notes

- **`consensus_score` is a Claude estimate**, not an independent gate — it measures *perspective
  convergence*, not correctness. `/wiki-review --strict` is the real fact-check.
- **No plugin dependency.** Orchestration is done by calling the Codex/Gemini CLIs directly, so
  there's nothing to break when a marketplace or plugin changes.
- **Provider auth changes.** Gemini's free OAuth tier can be blocked; switch to an API key
  (`GEMINI_API_KEY`) — see `GUIDE.md` §5.
- **Secrets stay local.** `.env` and the Obsidian plugin's `data.json` (which holds the API key)
  are `.gitignore`d. Don't commit them.

## License

[MIT](LICENSE) © 2026 bomsan69

*Original inspiration: "Your Vault as a Shared Brain" by Roan Brasil Monteiro. This
implementation was rebuilt from scratch (the article's repo was gone) and is plugin-free.*
