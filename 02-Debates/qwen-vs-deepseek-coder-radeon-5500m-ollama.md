---
title: "Debate — Radeon Pro 5500M 8GB + Ollama: qwen2.5-coder:7b vs deepseek-coder:6.7b, 메인은?"
date: 2026-07-05
type: debate
agents: [claude, codex, glm]
consensus_score: 85
status: reviewed
related: "[[small-open-weight-coding-models-intel-mac-amd-gpu]]"
reviewed_at: 2026-07-05
review_score: 90
review_fixes_applied: 2026-07-05
---

관련 리서치: [[small-open-weight-coding-models-intel-mac-amd-gpu]]. 해당 노트에서 Claude/Codex는 qwen2.5-coder를 1·2순위 모두, GLM은 qwen2.5-coder-7b(1순위)+deepseek-coder-6.7b(2순위, VRAM 적합성/생태계 성숙도 이유)를 추천해 2순위가 갈렸습니다. 확정된 하드웨어(Radeon Pro 5500M 8GB, 2019 16" MacBook Pro, Ollama)에서 **"메인 하나만 고른다면"**을 명시적으로 물어 이 긴장을 해소했습니다.

## Round 1 — 입장

### 🔵 Claude
**입장: qwen2.5-coder:7b가 메인이어야 한다. deepseek-coder:6.7b는 이 용도에 명백히 열등하다.**

결정적 이유는 학습 시점 격차 — DeepSeek-Coder(v1) 6.7B는 2023년 말~2024년 초 모델(Ollama 라이브러리 "2년 전 업데이트" 명시)인 반면 Qwen2.5-Coder는 2024년 하반기 출시로 최신 코드 벤치마크에서 훨씬 강력합니다. VRAM 여유는 이미 둘 다 확보됐으므로("qwen2.5-coder:7b도 4.7GB로 8GB에 완전히 들어감") GLM이 원래 든 "DeepSeek이 더 안전하게 들어간다"는 이유는 이 하드웨어에서 이미 해소된 문제이고, 남는 차이는 순수 코드 품질입니다.

### 🔴 Codex
**입장: qwen2.5-coder:7b.** 핵심은 "요즘 코딩 어시스턴트로서의 품질" — DeepSeek-Coder 6.7B는 한때 좋은 모델이었지만 세대가 오래됐고, "이 코드 왜 깨졌는지 설명", "테스트 추가", "함수 리팩터링" 같은 실제 메인 업무에서 Qwen이 낫습니다. 8GB VRAM 적합성에서도 Qwen이 불리하지 않고(4.7GB), Ollama 사용성·문서·커뮤니티 사례도 Qwen이 더 현대적입니다.

### 🟡 GLM
**입장: qwen2.5-coder:7b가 압도적으로 옳다.** DeepSeek의 최신 코딩 능력은 Coder-V2(16B MoE)로 이행했고 6.7B는 구세대에 머뭅니다. 8GB VRAM에서 둘 다 Q4_K_M 완전 오프로드가 가능하므로 메모리 적합성은 동일한 출발선이고, 그렇다면 더 높은 품질의 Qwen을 고르는 게 자명합니다. DeepSeek 6.7B의 "안정성·생태계 성숙도" 장점도 현재는 Qwen이 더 활발히 유지보수되며 역전됐습니다.

**Round 1 결과:** 세 참가자 모두 독립적으로 qwen2.5-coder:7b를 만장일치로 선택. (주목: GLM은 원본 리서치에서 DeepSeek을 2순위로 밀었으나, "메인 하나만" 질문엔 스스로 입장을 뒤집음.)

## Round 2 — 반박 (만장일치 자체를 공격)

### 🔵 Claude
**자기 반박:** 비교 자체가 불공정할 수 있습니다. 세 참가자 모두 "deepseek-coder:6.7b(구세대)"와 비교했는데, DeepSeek의 실제 최신 소형 코딩 모델은 **DeepSeek-Coder-V2-Lite**(16B MoE, 활성 ~2.4B, Ollama 다운로드 ~8.9GB)입니다. 이건 **8GB VRAM보다 다운로드 크기 자체가 큽니다** — 즉 우리 결론은 "DeepSeek이 근본적으로 열등해서"가 아니라 "DeepSeek의 최신 모델이 이 VRAM에 안 맞아서"일 수 있습니다. 또한 노트북 서멀 스로틀링은 모델 선택과 무관하게 둘 다 겪는 문제이고, 라이선스(Qwen Apache 2.0 vs DeepSeek 자체 라이선스)는 상업적 이용 시 확인이 필요합니다.

### 🔴 Codex
만장일치의 약점은 "DeepSeek = deepseek-coder:6.7b"로 고정한 것 — strawman에 가깝습니다. 공정한 상대는 DeepSeek-Coder-V2-Lite(16B MoE, active 2.4B, 160K context — Ollama 공식 페이지 기준, Ollama `deepseek-coder-v2:16b` 8.9GB)입니다. 문제는 다운로드 8.9GB가 이미 8GB VRAM보다 크고 KV cache/Metal 오버헤드까지 필요해 **완전 GPU 상주가 어렵다**는 것 — 부분 CPU offload로는 돌아가도 단일 메인으로는 qwen2.5-coder:7b의 4.7GB급 안정성과 비교가 안 됩니다. 열/전력 제한도 장시간 생성에서 모델 차이보다 체감 성능을 더 지배할 수 있어 "지속 tok/s와 팬 소음"을 실측해야 합니다. 라이선스는 Qwen2.5-Coder-7B-Instruct가 Apache-2.0으로 상업 사용·재배포에 깔끔한 반면, DeepSeek-Coder-V2-Lite는 `deepseek-license`(커스텀 조항 포함)라 제품 내장·재배포·회사 사용 시 Qwen이 법무 리스크가 낮습니다. deepseek-coder:6.7b가 완전히 죽은 선택은 아니며, FIM/infilling, 알고리즘식 Python/C++ 단발 완성, 짧은 autocomplete에서는 여전히 Qwen보다 덜 수다스럽고 직접적일 수 있습니다.

### 🟡 GLM
DeepSeek-Coder-V2-Lite가 진짜 경쟁자라는 데 동의하지만, **Codex와 다른 결론**: 이 모델은 16B MoE지만 활성 파라미터가 ~2.4B에 불과해 Qwen2.5-Coder 7B(전체 활성)보다 *적게* 연산합니다. 8GB VRAM에 전부 들어가진 않아도, 대부분 층이 VRAM에 올라가고 소량만 시스템 RAM으로 spill되면 활성 전문가(expert)만 GPU에 상주해 실제 속도 저하가 제한적일 수 있습니다. "구세대 DeepSeek"이라 일축한 건 V2-Lite를 회피한 strawman이며, V2-Lite는 DeepSeek-V2 체크포인트에서 추가 6조(6T) 토큰으로 further pre-training(총 노출 10.2조 토큰, arXiv 2406.11931)되어 다국어 코드·긴 컨텍스트에서 Qwen 7B와 경쟁 가능합니다. (원 발언의 "33B 토큰"은 오류 — `DeepSeek-Coder-33B`라는 별도 모델의 파라미터 수와 혼동한 것으로 확인됨.) 열 스로틀링 환경에서는 활성 파라미터 ~2.4B(V2-Lite) vs 7B(Qwen)의 차이가 토큰당 추론 비용에서 3배 가까운 격차로 나타나, 스로틀링이 겹치면 Qwen이 오히려 더 불리해질 수 있습니다. 라이선스는 상업 프로젝트라면 Qwen의 Apache 2.0이 안전하지만 비상업·개인 사용에서는 무의미합니다. DeepSeek 6.7B는 Python FIM 단순 자동완성과 C/C++ 저수준 코드에서 여전히 강할 수 있습니다.

## ✅ Consensus Synthesis

**메인 모델 결론은 두 라운드 모두 만장일치로 유지됐지만, 2라운드에서 새로운 미해결 이견이 생겼습니다 — 이를 숨기지 않고 명시합니다.**

1. **메인 추천은 확정: `qwen2.5-coder:7b`.** 세 모델 모두 1·2라운드 통틀어 이 결론을 유지했습니다. 최신 코드 벤치마크 우위, 8GB VRAM에 여유 있게 적합(4.7GB), Ollama 생태계 활발함, Apache 2.0 라이선스로 상업 사용 안전.
2. **원래 비교(vs deepseek-coder:6.7b)는 불공정했다는 데 세 모델 모두 동의(자기 수정).** 공정한 DeepSeek 비교 대상은 6.7B(구세대, 2년 전)가 아니라 **DeepSeek-Coder-V2-Lite**(16B MoE, 활성 ~2.4B, Ollama 다운로드 ~8.9GB)입니다.
3. **미해결 이견 (Codex vs GLM):** V2-Lite의 8.9GB 다운로드가 8GB VRAM을 초과할 때 —
   - **Codex:** 부분 CPU 오프로드가 필요해져 단일 메인으로 쓰기엔 불안정하다.
   - **GLM:** MoE 구조상 활성 파라미터가 ~2.4B뿐이라 일부 시스템 RAM으로 spill돼도 실제 속도 저하가 제한적일 수 있고, 열 스로틀링 상황에서는 오히려 Qwen(7B 전체 활성)보다 유리할 수 있다.
   
   **이 이견은 이 노트에서 해결되지 않습니다** — MoE 모델의 부분 오프로드 시 실제 처리량은 벤치마크 없이 이론만으로 단정하기 어렵습니다. `[VERIFY]`: 실제로 `deepseek-coder-v2:16b`를 이 하드웨어에서 실행해 tok/s를 측정하기 전까지는 열린 질문으로 남깁니다.
4. **노트북 서멀 스로틀링이 모델 선택보다 체감 성능을 더 좌우할 수 있다는 데 Claude·Codex 합의.** 2019 16" MacBook Pro는 지속 부하 시 GPU 클럭이 떨어지는 특성이 잘 알려져 있어, 어느 모델을 쓰든 "지속 tok/s와 팬 소음"을 실측하는 게 모델 품질 논쟁보다 실용적일 수 있습니다.
5. **라이선스:** Qwen2.5-Coder(Apache 2.0)가 DeepSeek 계열(커스텀 라이선스 조항)보다 상업적 이용·재배포에 법무 리스크가 낮다는 데 세 모델 합의. 개인 로컬 사용에는 무관.
6. **deepseek-coder:6.7b의 잔존 가치:** 메인으로는 부적합하지만, Python FIM(fill-in-the-middle) 자동완성이나 C/C++ 저수준 코드 단발 완성에서는 여전히 유용할 수 있다는 데 Codex·GLM 합의 — 보조/백업 모델로는 유지할 가치가 있습니다.

**실무 적용:** `qwen2.5-coder:7b`를 메인으로 설정하세요(`ollama pull qwen2.5-coder:7b`). `deepseek-coder:6.7b`는 삭제하지 말고 FIM/저수준 자동완성용 보조로 남겨두는 걸 권장합니다. 여유가 되면 `ollama pull deepseek-coder-v2:16b`를 직접 실행해 실제 tok/s를 재보는 것이 이 노트의 유일한 미해결 이견(3번)을 해소하는 방법입니다.
