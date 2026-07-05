---
title: "Review — LoRA vs QLoRA Fine-Tuning Tradeoffs"
date: 2026-07-04
type: review
agents: [claude, codex]
source: 01-Research/lora-vs-qlora-fine-tuning-tradeoffs.md
review_score: 82
mode: strict
status: reviewed
---

Strict review of [[lora-vs-qlora-fine-tuning-tradeoffs]] (consensus_score 88). Two reviewers (Claude, Codex) fact-checked every quantitative and factual claim. Verdict: **technically strong, but one real accuracy overstatement and one internal contradiction need fixing**, plus several vendor numbers that should be marked as unverifiable.

## 🔵 Claude — Reviewer Notes

**Core claims hold up.** Memory arithmetic is correct (fp16 ≈ params×2 B; NF4 ≈ params×0.5 B + metadata). QLoRA's three contributions (NF4, double quantization, paged optimizers) are stated correctly. Variant descriptions (DoRA magnitude/direction, rsLoRA `α/√r`, LoftQ joint init, GaLore gradient projection) are accurate. The "merge into fp16 base then re-quantize" serving caveat is correct and valuable.

**Overstated / optimistic:**
- **Unsloth "2–5x speedups and 50%+ VRAM savings"** — open-source single-GPU is closer to ~2x; upper end is Pro/multi-GPU marketing. `[VERIFY]`
- **vLLM/LoRAX "negligible latency overhead"** — multi-adapter serving overhead is small but *not zero*. `[VERIFY]`

**Code nits:**
- `bnb_4bit_compute_dtype="bfloat16"` passed as a **string**; `BitsAndBytesConfig` expects `torch.bfloat16`. Version-dependent warn/error.
- `meta-llama/Llama-2-7b-hf` is a **gated** repo (needs HF token) — note for reproducibility.

## 🔴 Codex — Reviewer Notes

- **Accuracy error:** "single 24 GB card, want a 33B–70B model → QLoRA is often the only option that fits." The QLoRA paper reports **65B on a single 48 GB GPU**, *not* 24 GB. 33B can fit ~24 GB; extending to 70B on 24 GB is misleading. (arxiv 2305.14314)
- **Internal inconsistency (contradicts the Synthesis's "no contradictions"):** 70B fit claims disagree across sections — Claude implies 33B–70B on 24 GB, Gemini says 70B on 48 GB, Codex says 70B training ≈ 48–80 GB.
- **QLoRA 4-bit table is approximate:** real NF4 storage includes quantization metadata; QLoRA reports 7B 4-bit base ≈ 5,048 MB and Guanaco-65B ≈ 41 GB — "base only" figures are implementation-dependent.
- **DoRA overstated:** "matching full fine-tuning capability" — the paper narrows the gap; it does not establish general equivalence to full FT. (arxiv 2402.09353)
- **Notation:** Codex writes `A @ B`, Claude writes `B·A`; harmonize the convention (both describe the same low-rank product but read as conflicting dimensionally).
- rsLoRA / LoftQ / GaLore descriptions are broadly correct; GaLore's "7B on 24 GB" is a headline setting, not a universal guarantee.

## Action Items

- **[FIX] Accuracy:** Correct the "33B–70B on a single 24 GB card" claim — QLoRA fits ~33B on 24 GB and 65B on 48 GB; 70B on 24 GB is wrong.
- **[FIX] Consistency:** Reconcile the 70B memory figures across all three sections and update the Synthesis line "converge cleanly with no contradictions" (there is one).
- **[FIX] Overstatement:** Soften DoRA "matching full fine-tuning capability" → "narrows the gap to full fine-tuning."
- **[FIX] Code:** `bnb_4bit_compute_dtype=torch.bfloat16` (not the string); note Llama-2 gating.
- **[FIX] Notation:** Use one convention for the low-rank product (`B·A` vs `A@B`) across sections.
- **[VERIFY] Unsloth "2–5x / 50%+ VRAM":** vendor/benchmark-dependent; not independently generalizable.
- **[VERIFY] vLLM/LoRAX "hundreds of adapters, negligible overhead":** S-LoRA reports small, non-zero overhead — qualify the claim.

## Summary

**review_score: 82/100.** The note is accurate on all the high-stakes mechanics (memory math, QLoRA internals, variant definitions, serving caveat) and is genuinely useful. It loses points on: (1) one factual overstatement (70B on a 24 GB card), (2) a self-consistency issue that directly undercuts the Synthesis's "no contradictions" claim, and (3) a handful of optimistic vendor numbers presented as fact. All are correctable in a short editing pass — none invalidate the note's conclusions. Recommended next step: apply the 5 `[FIX]` items, then this note is publish-grade for `04-Published/`.
