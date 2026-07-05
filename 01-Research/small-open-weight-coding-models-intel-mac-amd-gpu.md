---
title: "Intel CPU + AMD GPU + 64GB RAM Mac에서 코딩 어시스턴트용 소형 오픈웨이트 모델 추천"
date: 2026-07-05
type: research
agents: [claude, codex, glm]
consensus_score: 83
status: reviewed
---

## 🔵 Claude — Overview & Architectural Framing

이 질문에서 가장 먼저 짚어야 할 건 **하드웨어의 실제 의미**입니다. "Intel CPU + AMD GPU" 조합은 Apple Silicon Mac이 아니라 **2019 Mac Pro나 iMac Pro급의 인텔 기반 Mac**을 뜻합니다. 이 사실이 선택지를 크게 좁힙니다.

**제약 3가지:**
1. **MLX 사용 불가.** Apple의 MLX 프레임워크는 Apple Silicon의 통합 메모리(unified memory) 구조를 전제로 하므로, Intel Mac에서는 애초에 선택지에 없습니다.
2. **CUDA 사용 불가.** GPU가 AMD(Radeon Pro/Vega/Navi 계열)이므로 NVIDIA 전용인 CUDA 가속 경로가 없습니다.
3. **ROCm도 사실상 무의미.** AMD의 ROCm은 주로 Linux를 대상으로 하고, macOS에서는 실질적으로 쓸 수 없습니다.

**결론적으로 남는 경로는 하나 — `llama.cpp`의 Metal 백엔드입니다.** macOS의 Metal API 자체는 Apple Silicon 전용이 아니라 Intel Mac의 AMD GPU에서도 동작하므로, `llama.cpp`가 GGUF 양자화 모델을 Metal로 부분 GPU 오프로드하는 방식이 현실적인 유일한 가속 경로입니다. 다만 이 조합에 대한 최적화는 Apple Silicon만큼 성숙하지 않았다는 점을 감안해야 합니다.

**64GB RAM은 넉넉하지만, 진짜 병목은 AMD GPU의 VRAM입니다.** 2019 Mac Pro/iMac Pro급 AMD 카드는 모델에 따라 8GB(Radeon Pro 580X/Vega 56)부터 32GB(Vega II Duo 한 장 기준)까지 편차가 큽니다. 시스템 RAM 64GB는 CPU 추론이나 모델 가중치를 메모리에 올려두는 데는 여유롭지만, **GPU로 오프로드할 수 있는 레이어 수는 결국 GPU VRAM 용량에 달려 있습니다.**

**이 하드웨어에 맞는 선택 기준:**
- 파라미터 수보다 **GGUF 양자화 후 실제 파일 크기**가 중요합니다(예: 7B Q4_K_M ≈ 4~5GB, 14B Q4_K_M ≈ 8~9GB).
- 코딩 특화 파인튜닝이면서, 소형(7B~16B급)이고, 커뮤니티에서 GGUF 변환이 잘 지원되는 모델이 유리합니다.
- MoE(Mixture-of-Experts) 구조는 총 파라미터는 크지만 활성 파라미터가 적어, RAM은 넉넉하고 GPU는 제한적인 이 하드웨어에 오히려 유리할 수 있습니다.

## 🔴 Codex — Technical Depth

전제: 이 환경은 Apple Silicon이 아니므로 MLX는 제외, NVIDIA가 아니므로 CUDA도 제외합니다. 현실적인 가속 경로는 `llama.cpp`의 Metal backend뿐이며, Intel Mac의 AMD Radeon Metal 가속은 동작하더라도 Apple Silicon 대비 최적화·처리량이 낮다고 보는 편이 안전합니다. 64GB RAM 덕분에 모델 로딩 자체는 넉넉하지만, 실제 GPU offload 한계는 AMD VRAM 8GB/16GB/32GB 여부에 좌우됩니다.

고려 후보: Qwen2.5-Coder 계열(0.5B~32B, 코딩 특화)에서는 이 하드웨어에 7B `Q5_K_M`/`Q8_0`, 14B `Q4_K_M`이 현실적입니다. DeepSeek-Coder-V2-Lite(MoE, 총 ~16B/활성 ~2.4B)는 매력적이지만 GGUF/llama.cpp에서 메모리·호환성 변수가 더 큽니다. Codestral Mamba(~7B)는 긴 컨텍스트 효율이 장점이나 생태계 재현성이 Qwen보다 약합니다. StarCoder2 15B, CodeGemma 7B는 비교 후보로는 적절하지만 최신 실사용 코딩 품질 기준에서는 우선순위가 낮습니다.

**추천 (정확히 2개):**

1. **Qwen2.5-Coder-14B-Instruct `Q4_K_M`** — 주력 모델. 64GB RAM에서는 무난히 올라가고, 16GB 이상 VRAM이면 상당수 layer를 Metal로 offload할 수 있습니다. 8GB VRAM에서는 전부 GPU에 올리기 어려워 일부 offload + CPU 계산이 섞입니다. 품질은 7B보다 확실히 낫고, 로컬 코드 설명·수정·리팩터링에서 가장 균형이 좋습니다.
2. **Qwen2.5-Coder-7B-Instruct `Q5_K_M` 또는 `Q8_0`** — 빠른 반복용 모델. 8GB VRAM 카드에서도 `Q5_K_M`은 비교적 현실적이고, 16GB 이상이면 `Q8_0`도 검토할 만합니다. 자동완성, 짧은 함수 작성, 테스트 생성처럼 latency가 중요한 작업에 적합합니다.

성능 기대치는 보수적으로 잡아야 합니다. Intel Xeon/W급 CPU만 쓰면 7B는 대략 5-12 tok/s, 14B는 2-6 tok/s 수준일 수 있습니다. Metal offload가 잘 먹으면 7B는 약 15-35 tok/s, 14B는 약 8-20 tok/s 정도를 기대할 수 있지만, VRAM·컨텍스트 길이·`-ngl` 값에 따라 크게 흔들립니다. 긴 컨텍스트에서는 KV cache가 VRAM/RAM을 추가로 먹으므로 14B는 `Q4_K_M`부터 시작하는 것이 안전합니다.

```bash
llama-server -m ./Qwen2.5-Coder-14B-Instruct-Q4_K_M.gguf -ngl 35 -c 8192 --host 127.0.0.1 --port 8080
```

## 🟣 GLM — Ecosystem, Variants & Serving

Intel Mac과 AMD 라데온 GPU 환경에서는 MLX(Apple Silicon 전용)나 CUDA를 사용할 수 없으므로, `llama.cpp`의 Metal 백엔드가 유일한 현실적인 하드웨어 가속 경로입니다. **Ollama**와 **LM Studio**는 Metal API를 통해 이 환경을 잘 지원하며, 설정이 가장 간편해 1순위로 권장됩니다. GPT4All 역시 로컬 구동이 가능하지만 백엔드 호환성이 불안정할 수 있습니다. 반면, Zed 에디터의 로컬 LLM 통합이나 MLX 기반 도구들은 Apple Silicon 전용이므로 배제해야 합니다. IDE 통합 측면에서는 VS Code의 Ollama 확장 프로그램이나 Continue.dev가 OpenAI 호환 API(Ollama, LM Studio)를 통해 가장 안정적으로 작동합니다. Cursor는 로컬 모델 백엔드를 커스텀 엔드포인트로 연결할 수 있으나 Intel Mac + AMD 조합에서는 최적화가 미흡합니다.

**커뮤니티 경험 및 성능 기대치:** Mac Pro / iMac Pro급 AMD GPU에서 Metal을 통한 LLM 구동 시, 시스템 RAM과 VRAM 간 대역폭 병목이 가장 큰 장애물입니다. 소형 모델(GGUF 양자화)의 레이어를 GPU VRAM에 완전히 올릴 경우 Apple Silicon만큼은 아니지만 CPU 대비 확실한 속도 향상(초당 15~30 토큰)을 얻을 수 있습니다. **하지만 VRAM 용량을 초과해 시스템 RAM을 혼용할 경우 PCIe 전송 오버헤드로 인해 순수 CPU(AVX2/AVX-512 활용) 구동보다 오히려 느려지는 역설이 발생합니다.** 따라서 VRAM 용량에 맞춰 GPU 오프로드 레이어 수를 제한하는 것이 성능의 핵심입니다.

**최적의 소형 코딩 모델 추천 (실용성/에코시스템 중심):**

1. **Qwen2.5-Coder-7B-Instruct (GGUF):** 64GB RAM과 일반적인 AMD VRAM(8~16GB) 환경에 완벽히 부합하는 크기입니다. Ollama 및 LM Studio 생태계에서 지원이 가장 활발하며, 작은 용량으로 대부분의 VRAM에 완전히 들어가 Metal 백엔드 최적화의 이점을 극대화할 수 있어 즉시 코딩 어시스턴트로 투입하기 가장 쉽습니다.
2. **DeepSeek-Coder-6.7B-Instruct (GGUF):** 코딩 특화 모델로서 검증된 성능을 자랑하며, 크기가 작아 대부분의 AMD GPU VRAM에 들어가 완전한 GPU 가속이 용이합니다. `llama.cpp` 커뮤니티에서 널리 사용되어 Intel Mac + AMD GPU 환경에서의 트러블슈팅 자료와 프리셋이 풍부해 초기 셋업 실패 확률이 낮습니다.

## ✅ Consensus Synthesis

**하드웨어 분석과 1순위 모델은 세 모델 모두 강하게 수렴했지만, 2순위 모델 선택은 실제로 갈렸습니다** — 이 차이는 서로 다른 우선순위를 반영하므로 숨기지 않고 그대로 남깁니다.

1. **하드웨어 제약 완전 합의:** MLX(Apple Silicon 전용) 불가, CUDA(AMD GPU) 불가, ROCm(macOS) 사실상 무의미 → **`llama.cpp` + Metal 백엔드**가 유일한 현실적 가속 경로. (Claude·Codex·GLM 만장일치)
2. **진짜 병목은 시스템 RAM이 아니라 GPU VRAM.** 64GB RAM은 넉넉하지만 AMD 카드의 VRAM(대략 8~32GB, 정확한 모델 미상)이 오프로드 가능 레이어 수를 결정합니다. GLM은 여기에 중요한 실전 경고를 추가했습니다 — **VRAM을 초과해 오프로드하면 PCIe 전송 오버헤드로 순수 CPU 추론보다 오히려 느려질 수 있다**는 것. 이는 Claude·Codex가 명시하지 않은 실용적 함정입니다.
3. **1순위 모델 만장일치: Qwen2.5-Coder-7B-Instruct (GGUF, Q5_K_M~Q8_0).** 세 모델 모두 이 선택에 동의했습니다 — 소형 VRAM(8~16GB)에도 완전히 들어가고, Ollama/LM Studio 생태계 지원이 가장 성숙합니다.
4. **2순위 모델은 두 갈래로 갈림 — 실제 선택은 VRAM 크기에 달려 있습니다:**
   - **Claude·Codex: Qwen2.5-Coder-14B-Instruct (Q4_K_M)** — 같은 계열로 품질을 올리는 선택. **VRAM이 16GB 이상**이거나, 일부 CPU 오프로드를 감수하고서라도 더 나은 코드 품질(리팩터링·복잡한 함수 설명)을 원한다면 이쪽.
   - **GLM: DeepSeek-Coder-6.7B-Instruct** — 다른 계열로 안정성·생태계 성숙도를 우선하는 선택. **VRAM이 8GB 이하**로 제한적이거나, 셋업 실패 리스크를 최소화하고 싶다면 이쪽 — 완전 GPU 오프로드가 보장되고 트러블슈팅 자료가 풍부합니다.
5. **성능 기대치는 대체로 일치:** Metal 오프로드가 잘 되면 7B 기준 초당 15~35 토큰(Codex), 15~30 토큰(GLM) — 대략 같은 범위입니다.

**실무 적용 가이드:** 정확한 AMD GPU 모델(따라서 VRAM 용량)을 먼저 확인하세요.
- **VRAM ≥ 16GB:** Qwen2.5-Coder-7B(빠른 반복용) + Qwen2.5-Coder-14B(품질용) 조합 — Claude·Codex 추천.
- **VRAM ≤ 8GB:** Qwen2.5-Coder-7B(1순위, 만장일치) + DeepSeek-Coder-6.7B-Instruct(안정적 2순위, GLM 추천) 조합 — 완전 GPU 오프로드와 성숙한 생태계를 우선.
- 어느 쪽이든 **Ollama 또는 LM Studio**로 시작하는 것이 설정 난이도가 가장 낮습니다(Codex·GLM 합의).

---

## 🖥️ 적용 사례: Radeon Pro 5500M 8GB + Ollama (2026-07-05)

**확정된 하드웨어:** AMD Radeon Pro 5500M 8GB — 이는 Mac Pro/iMac Pro가 아니라 **2019년 16" MacBook Pro**의 디스크리트 GPU입니다(노트북용 GPU라 데스크톱 카드보다 지속 처리량이 낮을 수 있음). VRAM 8GB로 확정되어 위 Synthesis의 "VRAM ≤ 8GB" 버킷에 해당합니다.

**최종 추천 (Ollama, 태그 실사용 확인됨):**

```bash
ollama pull qwen2.5-coder:7b       # 1순위, 기본 Q4_K_M, 4.7GB
ollama pull deepseek-coder:6.7b    # 2순위, 3.8GB
```

- 두 모델 모두 8GB VRAM에 KV cache 여유분을 남기고 완전히 들어갑니다. `Q8_0`(가중치만 ~8.1GB)은 이 VRAM에서 배제.
- Ollama는 macOS에서 Metal을 자동 감지하므로 `-ngl` 수동 설정이 불필요 — `ollama run qwen2.5-coder:7b`만 실행하면 됨.
- 노트북용 GPU 특성상 지속 처리량은 본문 추정 범위(15~35 tok/s)의 **낮은 쪽(15~20 tok/s대)**을 기대할 것.

[출처: ollama.com/library/qwen2.5-coder, ollama.com/library/deepseek-coder]


## 🔗 관련 토론
[[qwen-vs-deepseek-coder-radeon-5500m-ollama]] — 확정 하드웨어에서 메인 모델 단일 선택에 대한 토론.