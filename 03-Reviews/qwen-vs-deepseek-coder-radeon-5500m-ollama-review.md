---
title: "Review — Radeon Pro 5500M 8GB + Ollama: qwen2.5-coder:7b vs deepseek-coder:6.7b 토론"
date: 2026-07-05
type: review
agents: [claude, codex]
source: 02-Debates/qwen-vs-deepseek-coder-radeon-5500m-ollama.md
review_score: 79
mode: strict
status: reviewed
---

Strict review of [[qwen-vs-deepseek-coder-radeon-5500m-ollama]] (consensus_score 85). 두 리뷰어(Claude — 웹 검증 포함, Codex)가 다운로드 크기·스펙·라이선스·MoE 논쟁을 확인했습니다. 결론: **핵심 결론(qwen2.5-coder:7b 메인)은 유효하고 대부분 수치가 정확하지만, GLM이 인용한 "33B 토큰 학습" 주장은 확인된 오류이며, 컨텍스트 윈도우도 부정확합니다.**

## 🔵 Claude — Reviewer Notes

**웹 검증 결과, 대부분의 수치는 정확합니다:**
- Ollama 다운로드 크기(qwen2.5-coder:7b 4.7GB, deepseek-coder:6.7b 3.8GB, deepseek-coder-v2:16b 8.9GB) — 확인됨.
- DeepSeek-Coder-V2-Lite 활성 파라미터 ~2.4B, 라이선스명 "deepseek-license" — [HuggingFace 공식 모델 카드](https://huggingface.co/deepseek-ai/DeepSeek-Coder-V2-Lite-Instruct)로 확인됨.

**확인된 오류 — GLM의 "33B 토큰 학습" 주장은 틀렸습니다.** DeepSeek-Coder-V2의 공식 논문(arXiv 2406.11931)에 따르면 DeepSeek-V2 체크포인트에서 **추가 6조(6T) 토큰**으로 further pre-training했고, 총 노출 토큰은 **10.2조(10.2T)**입니다. "33B"는 실제 수치보다 3자리 수 이상 작습니다 — **`DeepSeek-Coder-33B`라는 별도 모델의 파라미터 수(33B)와 혼동한 것**으로 보입니다. `[FIX]` 대상입니다.

**컨텍스트 윈도우도 부정확합니다.** 노트는 "128K context"라 썼지만, Ollama 공식 라이브러리 페이지의 `deepseek-coder-v2:16b`는 **160K**로 명시돼 있습니다. `[FIX]`

## 🔴 Codex — Reviewer Notes

- Ollama 다운로드 크기 3개 모두 정확함(각 페이지 확인). [ollama.com/library/qwen2.5-coder](https://ollama.com/library/qwen2.5-coder), [deepseek-coder](https://ollama.com/library/deepseek-coder), [deepseek-coder-v2](https://ollama.com/library/deepseek-coder-v2)
- "16B total / 2.4B active / 128K context"는 HF 카드와 대체로 일치하나, Ollama는 파라미터를 더 정확히 **15.7B**로 표시합니다(사소한 정밀도 차이).
- **"V2-Lite는 33B 토큰 학습"은 사실상 오류.** 공식 카드/논문은 "추가 6T 토큰"이라 설명 — `DeepSeek-Coder-33B`(파라미터명)와 혼동했을 가능성이 큽니다. (Claude와 독립적으로 동일 결론)
- 라이선스: Qwen2.5-Coder-7B-Instruct = Apache-2.0 확인. DeepSeek-Coder-V2-Lite-Instruct = `deepseek-license`(코드 저장소는 MIT이나 모델 사용은 Model License 대상, 상업 사용 지원 문구 명시). 노트의 "커스텀 조항"이라는 표현은 타당하나, "상업 사용 불가"로 오독되면 안 됨 — 노트는 이미 "리스크가 낮다"는 상대적 표현만 써서 이 함정은 피했습니다.
- **Codex vs GLM의 MoE 오프로드 논쟁은 이론만으로 확정 불가능함을 재확인.** MoE의 활성 파라미터는 연산량을 줄이지만, 전체 expert 가중치는 어딘가에 상주해야 하고 CPU/RAM에 있는 expert가 토큰별로 필요하면 전송/대역폭 병목이 생길 수 있습니다. 실제 속도는 offload granularity, Metal 메모리 압박, KV cache, expert routing에 좌우되어 **노트가 이 부분을 "미해결"로 남긴 건 정확한 판단이었습니다.**
- `[VERIFY]` "deepseek-coder-v2:16b가 8GB VRAM에서 단일 메인으로 불안정" — 실제 벤치마크 없이는 확인 불가(노트도 이렇게 처리함).
- `[VERIFY]` "deepseek-coder:6.7b가 FIM/저수준 C/C++에서 Qwen보다 낫다" — 공식 비교자료로 확인 불가, 경험칙 수준.

## Action Items

- **[FIX] "33B 토큰 학습" 삭제/정정:** 실제로는 "추가 6조 토큰, 총 10.2조 토큰 노출"(arXiv 2406.11931)로 수정. `DeepSeek-Coder-33B`와의 혼동 가능성 명시.
- **[FIX] 컨텍스트 윈도우 정정:** "128K" → Ollama 공식 페이지 기준 **160K**로 수정.
- **[정밀도 참고]** 파라미터 표기를 "16B"에서 Ollama 표시값인 "15.7B"로 더 정밀하게 바꿀 수 있으나, 필수 수정은 아님.
- **[유지]** MoE 오프로드 논쟁을 "미해결"로 남긴 판단, 라이선스를 "리스크가 낮다"로 상대적으로 표현한 것 — 둘 다 정확했으므로 그대로 유지.

## Summary

**review_score: 79/100.** 메인 모델 결론(qwen2.5-coder:7b)과 세 가지 Ollama 다운로드 크기, DeepSeek-Coder-V2-Lite의 활성 파라미터·라이선스명은 모두 정확합니다. 감점 요인은 GLM이 인용한 **"33B 토큰 학습"이 실제(6T 추가/10.2T 총합)와 3자리 수 이상 차이 나는 확인된 오류**라는 점과, 컨텍스트 윈도우(128K→160K) 부정확입니다. 둘 다 짧은 수정으로 해결 가능하며, 메인 모델 결론이나 MoE 오프로드 관련 미해결 이견 처리 방식은 검증 결과 오히려 정확했습니다.
