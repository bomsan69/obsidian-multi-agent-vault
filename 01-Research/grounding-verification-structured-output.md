---
title: "구조화 출력으로 LLM 답변의 근거(Grounding)를 강제 검증하는 방법"
date: 2026-07-05
type: research
agents: [claude, codex, glm]
consensus_score: 90
status: reviewed
related: "[[restaurant-cs-chatbot-rag-architecture]]"
---

관련 토론: [[restaurant-cs-chatbot-rag-architecture]] — 이 노트는 그 토론에서 세 모델이 합의한 "근거 검증 이중 게이트" 결론의 **구체적 구현 방법**을 다룹니다.

> 이 노트는 GLM(z-ai/glm-5.2, OpenRouter 경유)이 Gemini를 대체한 이후 첫 3-way 리서치입니다.

## 🔵 Claude — Overview & Architectural Framing

문서 기반 QA 챗봇에서 "문서에 없으면 답하지 마라"는 **시스템 프롬프트 지시 하나로는 충분하지 않습니다.** 지시는 확률적으로 지켜지는 것이지 보장되는 것이 아니기 때문입니다. 이 문제를 해결하는 아키텍처 원칙은 하나입니다 — **"모델이 알아서 거부하길 기대"하지 말고, "애플리케이션이 근거 없는 답변의 출고 자체를 막는" 구조**로 바꾸는 것입니다.

**핵심 메커니즘: 근거를 답변과 분리하지 말고, 답변의 일부로 강제하라.** 자유 텍스트로 "~~에 따르면 ...입니다"라고 답하게 하면, 그 인용이 진짜인지 검증하기 어렵습니다. 대신 **구조화 출력(structured output/JSON schema)**을 이용해 모델이 다음을 함께 반환하도록 강제합니다:

```json
{
  "answer": "영업시간은 오전 11시부터 오후 10시까지입니다.",
  "supported": true,
  "evidence": ["문서 §2: 영업시간은 11:00-22:00입니다."]
}
```

이렇게 하면 "답변"과 "근거"가 같은 스키마 안에 묶여, 서버가 **근거 필드만 따로 검증**할 수 있습니다.

**검증은 반드시 서버(애플리케이션) 레벨에서 일어나야 합니다.** 모델이 `supported: true`라고 자체 보고해도 그 자체를 신뢰해선 안 됩니다 — 이는 여전히 모델의 자기 보고이고, 모델이 근거를 그럴듯하게 지어내는 "citation hallucination"이 발생할 수 있기 때문입니다. 진짜 안전장치는 **evidence 필드의 텍스트가 실제 원본 문서에 존재하는지를 애플리케이션 코드가 독립적으로 확인**하는 것입니다(문자열 매칭 또는 경량 검증 모델). 검증에 실패하면 모델의 답변을 그대로 내보내지 않고 **고정된 거절 문구로 강제 대체**합니다.

**왜 이게 "이중 게이트"인가:** 1차 게이트(모델 스스로 "이건 근거 없다"고 판단해 거부)와 2차 게이트(서버가 근거 필드를 독립 검증)가 함께 있어야, 한쪽이 뚫려도(모델이 지시를 무시하거나, jailbreak당하거나) 다른 쪽이 막습니다. 이건 우리가 식당 챗봇 케이스([[restaurant-cs-chatbot-rag-architecture]])에서 세 모델이 독립적으로 도달한 결론이기도 합니다 — 이번 리서치는 그 결론의 구체적 구현 방법을 파고드는 것입니다.

## 🔴 Codex — Technical Depth

핵심은 모델에게 "답변"만 생성하게 하지 않고, 답변을 뒷받침하는 `doc_id`, `paragraph_id`, `evidence_text`를 JSON Schema로 강제 출력하게 만든 뒤, 서버가 그 근거를 독립적으로 검증하는 것입니다. OpenAI는 Structured Outputs 또는 function calling의 `strict: true` 스키마를 사용할 수 있고, Anthropic은 tool use로 동일한 구조의 인자를 강제할 수 있습니다. JSON mode만 사용할 경우 형식 보장은 약하므로, 반드시 서버 측 schema validation을 추가해야 합니다.

예시 스키마:

```json
{
  "answer": "string",
  "citations": [
    {
      "doc_id": "string",
      "paragraph_id": "string",
      "evidence_text": "string"
    }
  ]
}
```

서버 검증은 2단계입니다. 먼저 `doc_id`와 `paragraph_id`가 실제 검색 결과 컨텍스트에 포함되는지 확인합니다. 그다음 `evidence_text`가 해당 문단에 존재하는지 검사합니다. 가장 엄격한 방식은 exact match입니다. 문장 분리나 공백 차이 때문에 실패가 잦다면 정규화 후 fuzzy match를 적용합니다. 답변이 근거 문장을 단순 인용하지 않고 요약한 경우에는 경량 NLI 모델로 `evidence_text entails answer` 여부를 확인할 수 있습니다. 단, NLI는 최종 권위자가 아니라 보조 신호로 두는 편이 안전합니다.

검증 실패 시에는 모델 답변을 그대로 노출하지 않고, 고정 거절 응답으로 교체합니다: "제공된 문서에서 답변을 뒷받침하는 근거를 확인할 수 없습니다." 이 응답은 모델이 아니라 **서버 코드가 반환**해야 합니다.

```python
from jsonschema import validate
from rapidfuzz import fuzz

SCHEMA = {
    "type": "object",
    "required": ["answer", "citations"],
    "properties": {
        "answer": {"type": "string"},
        "citations": {
            "type": "array",
            "minItems": 1,
            "items": {
                "type": "object",
                "required": ["doc_id", "paragraph_id", "evidence_text"],
                "properties": {
                    "doc_id": {"type": "string"},
                    "paragraph_id": {"type": "string"},
                    "evidence_text": {"type": "string"}
                }
            }
        }
    }
}

REFUSAL = "제공된 문서에서 답변을 뒷받침하는 근거를 확인할 수 없습니다."

def normalize(s):
    return " ".join(s.lower().split())

def verify(output, context_index):
    validate(output, SCHEMA)

    for c in output["citations"]:
        key = (c["doc_id"], c["paragraph_id"])
        paragraph = context_index.get(key)
        if not paragraph:
            return False

        ev = normalize(c["evidence_text"])
        para = normalize(paragraph)

        if ev in para:
            continue
        if fuzz.partial_ratio(ev, para) >= 92:
            continue

        return False

    return True

def guarded_answer(model_output, context_index):
    return model_output["answer"] if verify(model_output, context_index) else REFUSAL
```

대표 실패 사례는 citation hallucination입니다. 모델이 실제 문서에 없는 `doc_id`나 문단 번호를 그럴듯하게 만들어내는 경우입니다. 대응은 간단합니다 — 모델이 낸 citation을 신뢰하지 말고, 검색기가 넘긴 **허용 citation 집합과만 대조**합니다. 허용 목록 밖의 문서 ID, 문단 번호, 근거 문장은 모두 실패로 처리합니다.

## 🟣 GLM — Ecosystem, Variants & Serving

2024년 이후 OpenAI의 Structured Outputs, Anthropic의 Tool Use 기반 JSON 모드, Google Gemini의 JSON Schema 응답 제어 기능 등 주요 LLM 벤더의 네이티브 구조화 출력 지원이 표준으로 자리 잡았습니다. 이를 활용하는 프레임워크도 빠르게 성숙했습니다. **Instructor**는 Pydantic 기반 타입 힌트로 스키마를 강제하고 검증 실패 시 자동 재시도하여 개발자 경험을 극대화하며, **Guardrails AI**는 스키마 외에 비즈니스 로직(검증 규칙)을 결합한 폴리시 검증을 제공합니다. **LangChain**의 `StructuredOutputParser` 역시 체인 내 통합성을 강조하며 널리 쓰입니다.

이중 게이트 패턴의 두 번째 단계인 NLI 기반 entailment(함의) 검증에는 경량 모델이 주로 활용됩니다. HuggingFace 기반 **DeBERTa-v3** 계열 NLI 모델이나, **Vectara**의 HHEM(Hallucination Evaluation Model) 등 API 기반 전용 평가 서비스가 대표적입니다.

이 패턴은 법률 계약서 검토, 의료 가이드라인 QA, 금융 리포팅 등 근거 부재 시 치명적인 피해가 발생하는 도메인에서 실무 채택이 활발합니다. 대안 접근으로는 Cohere 등의 RAG 네이티브 인용(Citation) 기능 활용이나, 온도를 낮춰 여러 번 샘플링 후 답변의 교집합을 취하는 self-consistency 체크 등이 있습니다.

엄격한 이중 검증은 일반적인 텍스트 요약, 창의적 글쓰기, 내부 아이디에이션 툴에서는 지연 시간(latency)과 토큰 비용을 증가시키는 오버엔지니어링입니다. 반면, 규제 산업(Regulated industries)의 팩트 기반 QA, 에이전트의 툴 호출 인자 생성, 법적 책임이 수반되는 자동화 워크플로우에서는 JSON 스키마 강제와 NLI 기반 근거 검증을 결합한 이 패턴이 필수적인 안전망 역할을 수행합니다.

## ✅ Consensus Synthesis

세 모델이 모순 없이 강하게 수렴했습니다:

1. **원칙 일치:** 모델의 "거부 지시 준수"를 신뢰하지 말고, 근거를 구조화 출력(JSON Schema)으로 강제해 **서버가 독립 검증**하는 것이 핵심 원칙입니다. (Claude·Codex 합의)
2. **구현 상세 일치:** `doc_id`/`paragraph_id`/`evidence_text` 형태의 스키마, exact match → fuzzy match → 경량 NLI 순의 단계적 검증, 실패 시 고정 거절 응답으로 서버가 강제 대체하는 구조 (Codex의 구체적 코드로 뒷받침됨)
3. **생태계가 이미 성숙:** Instructor, Guardrails AI, LangChain StructuredOutputParser 같은 기존 도구로 바로 구현 가능하며, NLI 검증엔 DeBERTa-v3나 Vectara HHEM 같은 기성 옵션이 있습니다. (GLM)
4. **적용 범위 판단:** 규제 산업(법률·의료·금융)이나 법적 책임이 따르는 워크플로우엔 필수, 일반 요약·창작 도구엔 과잉 엔지니어링이라는 명확한 구분 기준 제시. (GLM)
5. **citation hallucination이라는 공통 실패 사례:** 모델이 근거 자체를 지어내는 경우, 검색기가 넘긴 **허용 citation 집합과만 대조**해야 한다는 방어책에 Codex와 GLM 모두 동의(GLM은 Vectara HHEM 같은 전용 hallucination 평가 모델을 대안으로 제시).

**실무 적용:** 식당 챗봇 케이스([[restaurant-cs-chatbot-rag-architecture]])처럼 hallucination이 절대 금지인 시스템에는, 이 노트의 Codex 코드 예시를 그대로 참고해 (1) JSON schema 강제, (2) exact/fuzzy match 검증, (3) 검증 실패 시 서버가 고정 거절문으로 대체하는 구조를 구현하면 됩니다.
