**Released:** April 2, 2026 by Google DeepMind
**License:** Apache 2.0 (unrestricted commercial use, modification, redistribution)

---
## Model Variants
| Model | Type | Total Params | Active Params | Context Window |
|-------|------|-------------|---------------|----------------|
| E2B | Dense (edge) | ~2.3B | 2.3B | 256K |
| E4B | Dense (edge) | ~4.5B | 4.5B | 256K |
| 26B A4B | Mixture of Experts | 25.2B | ~3.8B | 256K |
| 31B | Dense | 30.7B | 30.7B | 256K |

---

## Memory Requirements

### 4-bit Quantization (Q4)

| Model         | RAM Required |
| ------------- | ------------ |
| E2B           | ~4 GB        |
| E4B           | ~5.5–6 GB    |
| 26B A4B (MoE) | ~16–18 GB    |
| 31B           | ~17–20 GB    |

### 8-bit Quantization

| Model         | RAM Required |
| ------------- | ------------ |
| E2B           | ~5–8 GB      |
| E4B           | ~9–12 GB     |
| 26B A4B (MoE) | ~28–30 GB    |
| 31B           | ~34–38 GB    |

### BF16/FP16 (full precision)

| Model         | RAM Required |
| ------------- | ------------ |
| E2B           | ~10 GB       |
| E4B           | ~16 GB       |
| 26B A4B (MoE) | ~52 GB       |
| 31B           | ~62 GB       |

---
## MoE Architecture Note
The 26B A4B uses Mixture of Experts (MoE) with 128 experts total, activating only ~3.8B parameters per inference pass. However, the **full 25.2B parameter model must still be loaded into memory** — the "active parameters" figure does not reflect memory requirements.

---

## Key Features

- Multimodal: text + image inputs on all sizes; audio support on edge models (E2B, E4B)
- 140+ language support
- Up to 256K token context window
- Native support for reasoning and agentic workflows

---

## Benchmarks

- **31B Dense**: #3 open model on Arena AI text leaderboard; 89.2% on AIME 2026 (no tools)
- **26B A4B**: #6 open model on Arena AI text leaderboard
- Gemma 3 27B comparison: 20.8% on AIME 2026

---

## Minimum Hardware Recommendations

- **E2B**: 4 GB RAM minimum (4-bit) — suitable for most consumer hardware
- **E4B**: 6 GB RAM minimum (4-bit) — suitable for 8 GB systems with headroom
- **26B A4B**: 16–18 GB RAM minimum (4-bit) — requires 16 GB+ system
- **31B**: 17–20 GB RAM minimum (4-bit) — requires 24 GB+ system recommended

---

## Sources
- [Gemma 4: Byte for byte, the most capable open models — Google Blog](https://blog.google/innovation-and-ai/technology/developers-tools/gemma-4/)
- [Gemma 4 model overview — Google AI for Developers](https://ai.google.dev/gemma/docs/core)
- [Gemma 4 Memory Requirements: Complete Hardware Guide 2026](https://www.gemma4.wiki/requirements/gemma-4-memory-requirements)
- [Gemma 4 RAM Requirements: Complete Hardware Guide 2026](https://www.gemma4.wiki/requirements/gemma-4-ram-requirements)
- [Google DeepMind Releases Gemma 4 With Four Model Sizes](https://aihaven.com/news/gemma-4-launches-april-2026/)
- [Gemma 4 Hardware Requirements — Oflight Inc.](https://www.oflight.co.jp/en/columns/gemma4-hardware-requirements-local-ai-spec-2026)
- [Gemma 4 — LM Studio](https://lmstudio.ai/models/gemma-4)
- [What Is Google Gemma 4? — WaveSpeedAI Blog](https://wavespeed.ai/blog/posts/what-is-google-gemma-4/)