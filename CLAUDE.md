## Vault Identity

This vault is a multi-agent knowledge base for technical research.
Primary language: Korean
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

Rule: Agents must never write to 05-Published/. The `04_korean/` zone holds Korean translations and is agent-writable.

## Output Format

Every agent-written note must start with this frontmatter:

---
title: ""
date: YYYY-MM-DD
type: research | debate | review
agents: [claude, codex, gemini]
consensus_score: 0-100
status: draft | reviewed | published
---

Each agent signs its own section in the body:

## 🔵 Claude — [section name]
[Claude's contribution]

## 🔴 Codex - [section name]
[Codex's contribution]

## 🟡 Gemini - [section name]
[Gemini's contribution]

## ✅ Consensus Synthesis
[only written if consensus_score >= 75]

## Session Start

At the start of each session:
1. Read 00-Inbox/ for pending items
2. Read the last 3 notes in 01-Research/ for context continuity
3. Check 02-Debates/ for unresolved debates (status: draft)

## Translation

The `/translate` command converts English tech notes into natural Korean
(see `.claude/commands/translate.md`).

- **Source:** any vault note (e.g. `05-Published/…`, `01-Research/…`) or raw English text.
- **Output:** saved to `04_korean/` with the **same filename** as the source.
- Meaning-first (의역 우선). Standard English tech terms (LLM, RAG, Docker, JSON, …) stay in
  English; everything else becomes idiomatic Korean. Markdown structure is preserved; only code
  comments inside code blocks are translated.
- Agents may write to `04_korean/` but must never write to `05-Published/`.