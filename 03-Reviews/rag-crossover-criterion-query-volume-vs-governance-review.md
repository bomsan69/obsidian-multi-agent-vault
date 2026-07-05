---
title: "Review — RAG 전환점의 1차 기준: 질의량인가, 거버넌스 요구사항인가?"
date: 2026-07-05
type: review
agents: [claude, codex]
source: 02-Debates/rag-crossover-criterion-query-volume-vs-governance.md
review_score: 72
mode: strict
status: reviewed
---

Strict review of [[rag-crossover-criterion-query-volume-vs-governance]] (consensus_score 90). 두 리뷰어(Claude, Codex)가 가격 수치·논리 구조·일관성을 검증했습니다. 결론: **핵심 통찰(거버넌스 게이트 개념)은 유효하지만, "vector-store가 강제된다"는 결론에 논리적 비약이 있고, 가격 수치 다수가 부정확하거나 확인되지 않았으며, "두 라운드 모두 같은 결론"이라는 synthesis의 자체 서술도 과장입니다.**

## 🔵 Claude — Reviewer Notes

**결정적 문제 — "거버넌스 요구사항 = vector-store 강제"라는 결론이 과장됨.** Synthesis는 ACL·provenance 요구가 있으면 질의량 무관 vector-store가 강제된다고 정리했지만, 이는 **"문서 단위 필터링/구조화가 필요하다"**와 **"그래서 임베딩 기반 유사도 검색이 필요하다"**를 혼동한 것입니다.

- **ACL:** 권한에 맞는 문서만 고르는 건 단순 규칙 기반 필터링(예: `WHERE owner_id = user.team_id`)으로 충족 가능하고, 필터링된 부분집합이 컨텍스트에 들어가면 여전히 prompt 삽입으로 처리 가능합니다. 벡터 유사도 검색은 "필터링 후에도 여전히 커서 의미 검색이 또 필요할 때"만 필요합니다.
- **출처 인용:** "문서 ID + 단락 번호"를 프롬프트에 그대로 넣는 방식으로도 해결 가능하며, 벡터 임베딩이 필수는 아닙니다.

이 debate의 세 참가자(저 자신 포함) 모두 이 구분을 놓쳤습니다. `[FIX]` 대상입니다.

제 가격 반박("Anthropic 캐시 쓰기 ≈125%")은 방향은 맞지만 5분/1시간 캐시별로 값이 다르다는 세부사항이 빠져 있었습니다.

## 🔴 Codex — Reviewer Notes

- **가격 — Claude 주장 (Anthropic 125%):** 부분적으로 정확. Anthropic 공식 가격표는 5분 캐시 쓰기 = 1.25배, 1시간 캐시 쓰기 = 2배, 캐시 읽기 = 0.1배로 명시. "쓰기=125%"만 일반화한 건 불완전.
- **가격 — Codex 자신의 R2 주장(Gemini $1.50/$0.15):** 현재 Gemini 공식 가격표와 정확히 일치하지 않음. 실제로는 모델별로 10% 또는 20%이며(Gemini 2.5 Pro 계열 20%, Flash/Lite 계열 10%), storage 시간당 비용이 별도로 붙음. "$1.50/$0.15" 조합은 확인되지 않음.
- **가격 — Codex 자신의 R2 주장(OpenAI gpt-5.5):** 현재 OpenAI 공식 API 가격표 기준으로 확인됨 — short context 입력 $5.00/캐시 $0.50(10%), long context 입력 $10.00/캐시 $1.00.
- **가격 — Gemini의 R2 주장("캐시 조회 비용 10~20%", "히트율 100%→최대 90% 할인"):** Gemini에 한해 대체로 맞지만 보편 법칙은 아님. 토큰 단가만 보면 10% 모델에서 가능하나, storage 비용이 별도로 붙어 **총비용 기준으로는 과장**될 수 있음.
- **내부 일관성:** 최종 "governance gate first, then query volume"은 2라운드 주장과는 논리적으로 맞지만, **"두 라운드 모두 같은 결론"이라는 서술과 consensus_score 90은 과장**입니다. 1라운드는 "질의량" 합의였고, 2라운드에서 "거버넌스 우선"으로 **프레이밍 자체가 바뀌었습니다** — 이건 정제(refinement)이지 "두 라운드 내내 일치"가 아닙니다.
- **기술 주장:** "lost-in-the-middle"은 타당(Liu et al.). ACL/multi-tenant/provenance는 "vector-store가 강제된다"보다 **"권한 필터링 가능한 retrieval/문서 단위 선택이 강제된다"**로 쓰는 게 더 정확 — 작은 코퍼스는 DB 필터링 후에도 prompt-stuffing이 가능함 (Claude의 지적과 독립적으로 일치).
- **[VERIFY]** "top-k 서빙만이 경제적으로 지속 가능"은 워크로드·지연시간 SLO·캐시 TTL·임베딩/검색 비용·답변 품질 요구에 따라 달라져 독립 확인 불가.

## Action Items

- **[FIX] 논리 비약 정정:** "ACL/provenance → vector-store 강제"를 "ACL/provenance → 문서 단위 필터링·구조화 강제(반드시 벡터 검색은 아니며, 작은 필터링 결과는 prompt 삽입으로도 처리 가능)"로 수정.
- **[FIX] Synthesis 서술 정정:** "두 라운드 모두 같은 결론"이라는 표현을 "1라운드는 질의량 합의, 2라운드에서 거버넌스 게이트로 프레이밍이 정제됨"으로 수정. consensus_score를 재산정 필요.
- **[FIX] Anthropic 캐시 가격:** "≈125%"를 "5분 캐시 쓰기=1.25배, 1시간 캐시 쓰기=2배, 읽기=0.1배(모두 근사치, 최신 가격표 확인 필요)"로 구체화.
- **[FIX] Gemini 캐시 가격 (Codex 자신의 수치):** "$1.50/$0.15" 조합을 삭제하거나 "모델별로 10~20% + 별도 storage 비용"으로 일반화.
- **[VERIFY] OpenAI gpt-5.5 가격:** Codex는 확인됐다고 주장하나 최신 공식 가격표로 재확인 권장 (모델명·가격 모두 시점에 따라 바뀔 수 있음).
- **[VERIFY] "top-k 서빙만이 경제적 지속가능"** — 조건부 주장이므로 일반화 표현 완화 필요.

## Summary

**review_score: 72/100.** 핵심 개념적 기여 — "거버넌스 게이트가 질의량보다 우선할 수 있다"는 통찰 — 는 유효하고 유용합니다. 다만 (1) "vector-store가 강제된다"는 결론에 실제로는 "문서 단위 필터링"과 "벡터 검색"을 혼동한 논리 비약이 있고, (2) 가격 수치 다수가 부정확하거나(Codex의 R2 Gemini 수치) 세부조건이 빠졌으며(Claude의 Anthropic 수치), (3) "두 라운드 모두 수렴했다"는 synthesis 자체 서술이 실제 진행 경과(1라운드 질의량 → 2라운드 거버넌스로 프레이밍 전환)를 과장했습니다. 모두 정정 가능한 수준이며 핵심 결론(거버넌스 요구사항이 있으면 질의량과 무관하게 문서 단위 처리가 필요하다는 것) 자체를 뒤집지는 않습니다.
