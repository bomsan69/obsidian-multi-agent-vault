# 검증하는 두 번째 뇌 — Obsidian Vault에 Claude·Codex·Gemini를 직접 붙여 만든 멀티 에이전트 지식 베이스 (플러그인 없이)

> *블로그 하나를 따라 하다 저장소가 사라졌고, 결국 더 단순하고 더 정직한 버전을 직접 만든 이야기.*
>
> 📦 코드: [github.com/bomsan69/obsidian-multi-agent-vault](https://github.com/bomsan69/obsidian-multi-agent-vault)

---

## TL;DR

- **문제:** LLM은 자기 맹점을 확신 있게 말한다. 한 모델만 믿으면 틀린 걸 그대로 발행하게 된다.
- **접근:** 같은 주제를 **Claude·Codex·Gemini 세 모델에 붙이고**, 각자의 답을 **Obsidian vault에 기록**하고, 발행 전 **review 단계로 검증**한다.
- **반전:** 이걸 자동화해 준다는 오케스트레이션 플러그인은 실전에서 깨졌다. 그래서 **플러그인을 걷어내고 CLI를 직접 호출**하는 방식으로 갔다. 더 단순하고 안 깨진다.
- **핵심 교훈:** 가치를 낸 건 "합의 점수(consensus score)"라는 숫자가 아니라, **사람이 읽는 `[VERIFY]`·`[FIX]` 목록을 뽑아내는 review 단계**였다.

---

## 1. 출발점: 따라 할 저장소가 사라졌다

시작은 어느 Medium 글이었다 — *"Your Vault as a Shared Brain: Obsidian Multi-Agent with Claude Octopus, Codex, and Gemini."* 아이디어는 매력적이었다:

> 어떤 언어 모델도 자기 맹점을 인정하지 않는다. 해법은 셋 중 하나를 고르는 게 아니라, **셋 모두를 같은 문제에 붙이고, 각자 뭐라 했는지 기록하고, 신뢰하기 전에 consensus gate를 적용하는 것**이다.

그런데 글에 걸린 GitHub 저장소가 **이미 삭제되어 있었다.** `git clone`이 404를 뱉었다. 템플릿도, 스크립트도 없었다.

역설적이게도 이 사고가 이 글의 첫 번째 교훈을 증명했다 — **모든 게 로컬 평문 `.md` 파일이면 저장소가 사라져도 잃을 게 없다.** 그래서 저장소 없이, 글의 *개념*만 가지고 vault를 처음부터 다시 지었다. 그 과정에서 원본 글의 여러 주장이 지금 현실과 다르다는 것도 알게 됐다.

---

## 2. 큰 그림: 세 개의 역할

시스템은 딱 세 가지 역할로 나뉜다. 이 분리를 붙잡고 있으면 나머지는 세부사항이다.

| 역할 | 담당 | "무엇을" |
|---|---|---|
| **기억(Memory)** | Obsidian Vault (`.md` 파일) | 무엇을 아는가 |
| **오케스트레이션(Orchestration)** | `/wiki-*` 커맨드 → Codex·Gemini CLI 직접 호출 | 어떻게 여러 관점을 모으는가 |
| **검증(Verification)** | `/wiki-review --strict` → `[VERIFY]`/`[FIX]` | 무엇을 믿어도 되는가 |

원본 글은 여기에 "Octopus 플러그인"이라는 네 번째 요소를 오케스트레이션 자리에 넣었다. **우리는 그걸 뺐다.** (§9에서 왜인지 자세히)

---

## 3. Obsidian + Local REST API — 그리고 원본 글이 틀린 부분

Obsidian은 로컬 Markdown 에디터다. vault는 그냥 폴더고, 노트는 그냥 `.md` 파일이다. AI 에이전트가 파일을 읽고 쓸 수 있다는 뜻이다.

에이전트가 쓴 노트를 Obsidian이 **즉시** 반영하게 하려면 **Local REST API** 플러그인(coddingtonbear / Adam Coddington)을 쓴다. 여기서 원본 글과 결정적으로 갈린다.

> ⚠️ **원본 글은 "기본적으로 인증이 없다"고 했다. 지금은 틀렸다.**
> 플러그인은 v4.x에서 **"Local REST API with MCP"**로 바뀌었고, **모든 `/vault/**` 요청에 API 키(Bearer 토큰)가 필수**다.

내가 실제로 확인한 값:

```json
{"status":"OK","service":"Obsidian Local REST API",
 "versions":{"obsidian":"1.12.7","self":"4.1.3"},
 "authenticated":true}
```

- 기본 보안 포트: **HTTPS `27124`** (자체서명 인증서 → `curl -k`)
- 평문 포트: **HTTP `27123`** (옵션)
- 키 발급: Obsidian → Settings → Local REST API → **API Key 복사**

키 없이 `/`를 부르면 `"authenticated":false`만 나온다 — 서버가 살아있다는 것만 증명할 뿐, 쓰기가 된다는 뜻이 아니다. 진짜 확인은 키를 넣고 목록을 부르는 것:

```bash
export OBSIDIAN_API_KEY="여기에_플러그인이_준_키"
curl http://localhost:27123/vault/ -H "Authorization: Bearer $OBSIDIAN_API_KEY"   # 401이면 키 문제
```

노트 저장(생성/덮어쓰기):

```bash
curl -X PUT http://localhost:27123/vault/01-Research/my-note.md \
  -H "Authorization: Bearer $OBSIDIAN_API_KEY" \
  -H "Content-Type: text/markdown" \
  --data-binary @local-file.md
```

---

## 4. 키는 `.env`에, 슬래시 커맨드는 셸에서 로드

키를 커맨드 파일에 하드코딩하면 GitHub 공개 시 그대로 유출된다. 그래서 `.env`에 넣고 `.gitignore`로 막는다.

```bash
# .env  (git에 올리지 않음)
OBSIDIAN_API_KEY=...
GEMINI_API_KEY=...
```

```bash
# .gitignore
.env
.obsidian/workspace*.json
```

**함정 하나:** 슬래시 커맨드 파일은 셸이 아니라 프롬프트 텍스트다. `$OBSIDIAN_API_KEY`라고 써도 `.env`를 자동으로 안 읽는다. 게다가 이 실행 환경의 Bash는 호출 간 env가 유지되지 않는다. 그래서 **키 로드와 curl을 같은 셸 명령 안에서** 해야 한다:

```bash
KEY=$(grep -E '^OBSIDIAN_API_KEY=' .env | cut -d= -f2-)
curl -H "Authorization: Bearer $KEY" http://localhost:27123/vault/...
```

---

## 5. Vault 구조와 CLAUDE.md — 에이전트가 지키는 계약

```text
OctopusVault/
├── CLAUDE.md            # 에이전트 계약 — 모든 세션이 먼저 읽음
├── 00-Inbox/            # 원자료 (사람만)
├── 01-Research/         # /wiki-research 출력 (에이전트)
├── 02-Debates/          # /wiki-debate 출력 (에이전트)
├── 03-Reviews/          # /wiki-review 출력 (에이전트)
├── 04_korean/           # /translate 출력 — 한국어 번역 (에이전트)
├── 05-Published/        # 최종 큐레이션 (사람만)
└── .claude/commands/    # 슬래시 커맨드 정의 (vault 루트)
    ├── wiki-research.md
    ├── wiki-debate.md
    ├── wiki-review.md
    └── translate.md
```

```bash
mkdir -p 00-Inbox 01-Research 02-Debates 03-Reviews 04_korean 05-Published .claude/commands
```

`CLAUDE.md`는 계약서다. Claude Code는 작업 디렉터리의 `CLAUDE.md`를 세션 시작 때 자동으로 읽는다. 여기에 **쓰기 존(write zone)**, **출력 포맷(frontmatter + 서명 섹션)**, **세션 시작 규칙**을 못 박아 둔다.

```markdown
## Write Zones
| Zone      | Path         | Who writes |
|-----------|--------------|------------|
| Inbox     | 00-Inbox/    | Human only |
| Research  | 01-Research/ | All agents |
| Debates   | 02-Debates/  | All agents |
| Reviews   | 03-Reviews/  | All agents |
| Korean    | 04_korean/   | All agents |
| Published | 05-Published/| Human only |

Rule: 에이전트는 05-Published/ 에 절대 쓰지 않는다.
```

**골든 룰:** 에이전트는 `01`~`03`, `04_korean`에만 쓴다. `05-Published/`로 옮기는 건 사람만 한다. 이 분리가 "검증 안 된 내용이 발행되는 것"을 막는다.

> 💡 폴더 번호가 `04_korean` 다음 `05-Published`로 건너뛰는 건 실수가 아니다. 번역 레이어가 `04` 슬롯을 차지하도록 발행 폴더를 `05`로 옮겼다. 코드·문서·폴더 이름을 **한 값으로 통일**하는 게 이 시스템에서 반복적으로 중요했다.

---

## 6. 네 개의 슬래시 커맨드

### `/wiki-research <topic>` — 병렬 리서치

세 모델에 주제를 분배하고, 서명된 섹션으로 노트를 만들어 `01-Research/`에 저장한다. **핵심은 이게 플러그인이 아니라 CLI를 직접 부른다는 점이다:**

```markdown
3. 각 프로바이더에 병렬로 분배 (CLI 직접 호출):
   - Claude → 개요/아키텍처 섹션 (직접 작성)
   - Codex  → 기술 심화: `codex exec --skip-git-repo-check "…" < /dev/null`
   - Gemini → 생태계: `gemini --skip-trust -p "…" < /dev/null` (GEMINI_API_KEY)
4. 섹션 간 수렴도를 0-100으로 채점 → consensus_score.
   이건 **Claude의 추정치**이지 독립 알고리즘이 아니다.
   실제 사실검증은 뒤의 /wiki-review --strict 가 담당한다.
```

노트 frontmatter는 계약대로:

```yaml
---
title: "..."
date: 2026-07-04
type: research
agents: [claude, codex, gemini]
consensus_score: 88
status: reviewed        # consensus_score >= 75 일 때만 "reviewed"
---
```

본문은 각 모델이 자기 섹션에 서명한다: `## 🔵 Claude — …`, `## 🔴 Codex — …`, `## 🟢 Gemini — …`, 그리고 수렴 시 `## ✅ Consensus Synthesis`.

### `/wiki-debate <topic>` — 구조화된 토론

각 모델이 **명시적 입장**을 취하고 서로 **반박(rebuttal)**한다. 여기서 맹점이 드러난다. 관련 리서치 노트가 있으면 컨텍스트로 끌어오고 양방향 백링크를 건다.

### `/wiki-review <path> [--strict]` — 검증 (이 시스템의 심장)

기존 노트를 Claude+Codex로 리뷰한다. `--strict`면 **독립적으로 확인 불가한 주장마다 `[VERIFY]`를 붙이고**, 실제 오류엔 `[FIX]`를 단다. 원본 노트의 frontmatter에 `review_score`를 기록한다(본문은 안 건드림).

### `/translate <path>` — 한국어 번역 레이어 (내가 추가한 것)

원본 블로그엔 없던 기능이다. 영문 기술 노트를 자연스러운 한국어로 번역해 `04_korean/`에 같은 파일명으로 저장한다. **의역 우선, 업계 표준 영문 용어(LLM·RAG·PEFT·VRAM 등)는 보존, 코드는 그대로 두되 코드 주석만 번역.** 로컬 파일이라 이렇게 vault를 *내게* 맞게 확장할 수 있었다.

---

## 7. 실전에서 실제로 벌어진 일

이론은 깔끔하다. 현실은 아니었다. 이 시스템의 진짜 가치는 아래 두 사건에서 드러났다.

### 사건 1 — Gemini 무료 티어가 막혔다

어느 날 `gemini` 호출이 전부 실패했다:

```
IneligibleTierError: This client is no longer supported for
Gemini Code Assist for individuals.
```

재로그인해도 소용없었다. 로그인 문제가 아니라 **무료 OAuth 티어 자체가 차단된 것**이었다(`UNSUPPORTED_CLIENT`). 해법은 OAuth를 버리고 **API 키 방식**으로 가는 것:

```bash
# 1) Google AI Studio에서 API 키 발급 → .env 에 GEMINI_API_KEY
# 2) 인증 방식을 api-key로 전환
#    ~/.gemini/settings.json:
#    "security": { "auth": { "selectedType": "gemini-api-key" } }
# 3) 셸 프로필에 export (하위 프로세스가 상속하도록)
export GEMINI_API_KEY=...   # ~/.zshrc
```

교훈: **"공짜 OAuth로 3개 모델"이라는 원본 글의 전제는 생각보다 취약하다.** 프로바이더 인증은 언제든 바뀔 수 있다. 설계가 이 실패를 흡수할 수 있어야 한다.

### 사건 2 — Consensus 게이트가 진짜 오류를 잡았다

LoRA vs QLoRA 리서치 노트에서, 내가(Claude가) 확신을 갖고 이렇게 썼다:

> *"24 GB 카드 한 장으로 33B–70B 모델 → QLoRA만이 유일한 선택."*

`/wiki-review --strict`를 돌리자 **Codex가 QLoRA 논문을 인용하며 잡아냈다:**

> *"틀렸다. QLoRA 논문은 65B를 **48 GB**에서 학습했다. 24 GB가 아니다. 70B를 24 GB로 확장하는 건 오해다." [arxiv 2305.14314]*

이게 이 시스템 전체에서 **가장 가치 있던 순간**이다. 단일 모델이었다면 그 오류는 그대로 발행됐을 것이다. 그리고 주목할 점 — 오류를 잡은 건 "88점"이라는 숫자가 아니라 **리뷰어의 구체적인 지적**이었다.

---

## 8. Consensus 점수의 불편한 진실

원본 글은 "role-weighted consensus gate가 75+ 점이면 통과"라고 했다. 멋지게 들린다. 하지만:

1. **점수는 진실이 아니라 관점의 수렴을 잰다.** 세 모델이 같은 틀린 데이터로 학습했으면, 사이좋게 함께 틀린다. 원본 글도 §12에서 이걸 인정한다.
2. **플러그인 없이 우리 방식에선 그 점수를 Claude가 매긴다.** 심판이 선수 중 하나인 셈이라 객관성이 약하다.

그래서 우리는 점수를 **정직하게 `(estimated)`로 라벨링**한다. 그리고 진짜 안전장치는 숫자가 아니라 `/wiki-review --strict`가 뽑는 **사람이 읽는 `[VERIFY]` 목록**이라는 걸 명시한다.

> **핵심 통찰:** 이 시스템에서 믿을 건 "합의했다"가 아니라 "무엇을 확인해야 하는지 목록으로 나왔다"이다.

---

## 9. 왜 Octopus 플러그인을 걷어냈나

원본 글의 중심엔 `/octo:*` 오케스트레이션 플러그인이 있다. 우리도 제대로 설치하고, 진짜 게이트(`orchestrate.sh`)를 실행해 봤다. 결과:

- **런타임이 없어서** `/octo:setup`으로 부트스트랩해야 했다 (복잡성 +1).
- 실제 리서치를 돌리자 **Gemini 에이전트가 exit 137(killed)로 중단**됐다.
- 결정적으로, 플러그인이 에이전트 출력을 `trust="untrusted"` 프레임으로 감싸 synthesizer에 넘겼는데, **그 모델이 자기 입력을 프롬프트 인젝션으로 오인하고 합성을 거부**했다. → **최종 산출물 없음, 점수 없음.**

즉 플러그인의 자동 게이트는 **정작 필요한 순간에 아무것도 주지 못했다.** 반면 우리의 직접 CLI 호출 방식은 매번 안정적으로 작동했다.

**결정: 플러그인을 완전히 제거했다.** (uninstall + 런타임 삭제 + 마켓플레이스 등록 해제)

| | 플러그인 있이 | 플러그인 없이 (우리 방식) |
|---|---|---|
| 멀티모델 관점 | ✅ | ✅ (CLI 직접) |
| review `[VERIFY]` 검증 | ✅ | ✅ |
| vault 기억 | ✅ | ✅ |
| 독립 consensus 점수 | 이론상 ✅ (실측 실패) | ❌ (`estimated` 명시) |
| 단순함 / 안정성 | ❌ 복잡·취약 | ✅ |

잃은 건 "이론적으로 독립적인 점수" 하나뿐이고, 그건 실전에서 이미 못 믿을 것으로 판명됐다.

---

## 10. 세션을 넘어서는 기억

LLM의 근본 약점 — 세션마다 처음부터 시작 — 은 vault가 구조적으로 해결한다. 내일 새 세션을 열면 `CLAUDE.md`가 다시 이렇게 지시한다:

```markdown
## Session Start
1. 00-Inbox/ 의 대기 항목 확인
2. 01-Research/ 최근 3개 노트 읽어 컨텍스트 이어가기
3. 02-Debates/ 의 미해결 토론(status: draft) 확인
```

새 세션은 지난 대화를 "기억"하지 못한다. 하지만 **vault에 기록된 모든 작업에 접근**할 수 있다. 이게 휘발성 episodic 기억과 영속 semantic 기억의 차이다. 우리는 후자를 쓴다.

---

## 11. 한계와 교훈

- **토큰 비용.** 세 모델 병렬은 소비를 대략 3배로 만든다. 일상 리서치엔 `--agents=claude`, 중요한 결정에만 3-way.
- **게이트는 정확성을 보장하지 않는다.** 합의 ≠ 정답. `[VERIFY]` 항목은 1차 출처로 확인해야 한다.
- **Obsidian이 켜져 있어야** REST API가 작동한다. 닫혀 있으면 커맨드는 파일 직접 쓰기로 fallback하고, 노트는 재실행 시 나타난다.
- **프로바이더 인증은 변한다.** Gemini 사건처럼. `.env` + 셸 export로 인증 방식을 갈아끼울 수 있게 해두자.
- **정직한 라벨링이 시스템을 신뢰 가능하게 만든다.** 점수를 `(estimated)`로, 미검증 주장을 `[VERIFY]`로 드러내는 것이 "82/100 PASS"라는 가짜 확신보다 훨씬 유용하다.

---

## 12. 마무리

원본 블로그가 주장하고 싶었던 핵심은 **"AI를 검증 없이 믿지 말고, 그 불신을 도구로 만들라"**였다. 그 정신은 옳았다. 다만 그걸 구현하는 데 특정 플러그인은 필요 없었고, 오히려 방해가 됐다.

우리가 남긴 건 더 단순하다:
- **Vault = 기억** (로컬 `.md`, 락인 없음)
- **`/wiki-*` + CLI 직접 호출 = 여러 관점** (플러그인 의존성 없음)
- **`/wiki-review --strict` = 검증** (진짜 가치가 여기서 나온다)
- **`/translate` = 내게 맞춘 확장** (한국어 레이어)

저장소가 사라져도, 플러그인이 깨져도, 프로바이더 인증이 막혀도 — vault는 당신 디스크에 평문으로 남아 있다. 그게 이 접근의 진짜 힘이다.

---

## 코드와 저장소

전체 코드(`.claude/commands/`, `CLAUDE.md`, 셋업 스크립트, 이 vault 템플릿)는 아래에서 공개할 예정입니다:

```bash
git clone https://github.com/bomsan69/obsidian-multi-agent-vault.git
```

- 저장소: `https://github.com/bomsan69/obsidian-multi-agent-vault`
- 이슈/제안: `https://github.com/bomsan69/obsidian-multi-agent-vault/issues`
- 라이선스: `MIT` (see [LICENSE](LICENSE))

> 이 vault는 특정 플러그인이나 마켓플레이스에 의존하지 않습니다. Obsidian + Local REST API 플러그인 + Claude Code + (선택) Codex·Gemini CLI만 있으면 재현됩니다.

---

*이 글은 Claude Code와 함께, 실제로 vault를 처음부터 재구성하며 겪은 내용을 바탕으로 작성했습니다. 원본 영감: "Your Vault as a Shared Brain" (Roan Brasil Monteiro).*
