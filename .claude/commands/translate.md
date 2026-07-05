---
command: translate
description: Translate an English tech note into natural Korean (ko-tech-translator), saved into 04_korean/
argument-hint: "<vault-path>" | "<raw English text>"
---

# /translate — English → Korean Technical Translation

Translate AI / programming / IT content into natural Korean for Korean developers and
researchers. **Meaning-first (의역 우선), not literal.** Preserve industry-standard English
terms; render the rest as idiomatic Korean.

## 1. Resolve the source

1. Read `CLAUDE.md` and follow its Write Zones rules. **Never write to `05-Published/`** (human only).
2. Parse `$ARGUMENTS`:
   - If it looks like a vault path (e.g. `05-Published/foo.md`, `01-Research/bar.md`), **fetch** it via
     the Local REST API, loading the key from `.env` in the SAME shell command:
     `KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-); curl -H "Authorization: Bearer $KEY" http://localhost:27123/vault/<path>`
   - Otherwise treat `$ARGUMENTS` as raw English text to translate directly.

## 2. Translation contract

**Meaning-centered:** rebuild each paragraph as a Korean sentence; don't mirror English syntax.
Turn passive voice / long noun phrases / relative clauses into active, shorter Korean clauses.
Split over-long sentences. Drop unnecessary subjects (Korean omits them naturally).

**Keep these terms in English** (already standard in Korean dev community):
- AI/ML: LLM, SLM, RAG, RLHF, Fine-tuning, Prompt Engineering, Zero-Shot, Few-Shot,
  Chain-of-Thought, Reasoning, Embedding, Token, Tokenizer, Inference, Training, Hallucination,
  Grounding, Alignment, Agent, Multi-agent, Tool use, Function calling
- Models/products: GPT-4o, Claude, Gemini, Llama, Mistral, Qwen, Stable Diffusion
- Paradigms: OOP, FP, MVC, REST, GraphQL, gRPC, WebSocket, OAuth, JWT
- Tooling: Git, Docker, Kubernetes, CI/CD, DevOps, MLOps, Terraform
- DS/algo: hash map, binary tree, heap, queue, stack, O(n), Big-O
- Cloud/infra: VPC, IAM, S3, EC2, Lambda, CDN, DNS, TLS/SSL
- Formats: JSON, YAML, TOML, CSV, Parquet, Protobuf, Markdown
- Languages/frameworks: Python, TypeScript, Rust, React, Next.js, FastAPI, LangChain

**Translate these to Korean** (English in parens when helpful on first use):
deploy→배포, build→빌드, release→릴리스, repository→저장소, branch→브랜치, commit→커밋,
merge→병합, pull request→풀 리퀘스트(PR), latency→지연 시간(latency), throughput→처리량(throughput),
scalability→확장성, concurrency→동시성, asynchronous→비동기, dependency→의존성,
refactoring→리팩토링, context window→컨텍스트 윈도우, benchmark→벤치마크, dataset→데이터셋,
parameter→파라미터, architecture→아키텍처.

> Rule of thumb: if Korean tech blogs (velog, 네이버 기술블로그) keep the term in English, keep it;
> otherwise translate and put English in parentheses on first mention.

**Tricky English constructs:** "It is worth noting"→"주목할 점은", "under the hood"→"내부적으로",
"out of the box"→"별도 설정 없이/기본 제공", "opinionated"→"특정 방식을 강제하는",
"battle-tested"→"실전 검증된", "drop-in replacement"→"바로 교체 가능한 대안",
"boilerplate"→"보일러플레이트", "first-class citizen"→"일급 객체".

**Tone:** formal `~합니다/~입니다` for docs/papers; `~해요/~입니다` mix acceptable for blogs/tutorials.
Prefer active voice. Match the source's register.

## 3. Structure & fidelity rules

- Preserve the original **Markdown structure** exactly: headings, lists, tables, code blocks, frontmatter.
- **Do not translate code** inside code blocks; **do translate comments** inside code to Korean.
- Do not add content the source doesn't have. If a translator's note is truly needed, use
  `> **역주**: ...`.
- Abbreviations: spell out (English) on first use — e.g. `대규모 언어 모델(LLM)` — then use the abbreviation.
- If the source has YAML frontmatter, keep it; you may translate the `title` value but leave keys/enums intact.

## 4. Save

Derive `<basename>` = the source filename (for raw text, let the user name it or slugify the title).
Save the Korean translation via REST API PUT to `04_korean/<basename>`, loading the key from `.env`
in the same shell command:

KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-)
curl -X PUT http://localhost:27123/vault/04_korean/<basename> \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: text/markdown" \
  --data-binary @<tmpfile>

If the REST API is unreachable, fall back to writing `04_korean/<basename>` directly and tell the user
to reopen Obsidian.

## 5. Report

`Translated <basename> → 04_korean/<basename>` and, if asked, a short note of the tricky terms/decisions.

Never write to `05-Published/`.
