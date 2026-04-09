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

## Hardware Requirements

### 4-bit Quantization (Q4)
| Model | RAM | GPU VRAM | CPU-Only | TPS (GPU) |
|-------|-----|----------|----------|-----------|
| E2B | ~4 GB | 4–6 GB | Yes | ~17 |
| E4B | ~5.5–6 GB | 6–8 GB | Yes (slower) | ~14 |
| 26B A4B | ~16–18 GB | 16–18 GB | Not recommended | ~10 |
| 31B | ~17–20 GB | 20–24 GB | Not recommended | ~0.5* |

### 8-bit Quantization
| Model | RAM | GPU VRAM | CPU-Only | TPS (GPU) |
|-------|-----|----------|----------|-----------|
| E2B | ~5–8 GB | 5–8 GB | Yes | ~17 |
| E4B | ~9–12 GB | 9–12 GB | Yes (slower) | ~14 |
| 26B A4B | ~28–30 GB | 28–30 GB | Not recommended | ~10 |
| 31B | ~34–38 GB | 34–38 GB | Not recommended | ~0.5* |

### BF16/FP16 (full precision)
| Model | RAM | GPU VRAM | CPU-Only | TPS (GPU) |
|-------|-----|----------|----------|-----------|
| E2B | ~10 GB | 10 GB | Yes | ~17 |
| E4B | ~16 GB | 16 GB | Yes (slower) | ~14 |
| 26B A4B | ~52 GB | 52 GB | Not recommended | ~10 |
| 31B | ~62 GB | 62 GB | Not recommended | ~0.5* |

> *31B TPS figure likely reflects a CPU-only benchmark. GPU performance would be significantly higher. TPS benchmarks sourced from community testing on 48 GB VRAM GPU hardware.

---

## What "Effective Parameters" Means (E2B / E4B)

The "E" in E2B and E4B stands for **Effective** — as in, the number of parameters actively used during inference. Despite being labeled 2B and 4B, the actual stored parameter counts are higher. Like the 26B A4B, these models use a sparse architecture so not all weights are activated per token.

However, E2B and E4B use a different efficiency mechanism than MoE: **Per-Layer Embeddings (PLE)**. Instead of routing tokens to different expert networks, PLE adds a small dedicated conditioning vector for each token at every layer. This lets each decoder layer receive token-specific information only when relevant, without the overhead of full expert routing.

They also use a **hybrid attention** mechanism — interleaving local sliding window attention (fast, low memory) with full global attention (needed for long-range understanding). The final layer is always global. This gives the models the speed and memory footprint of a small model without losing long-context capability.

In short: E2B/E4B are **not naive small dense models**. They're architecturally **optimized for edge/on-device deployment** while punching above their weight in quality.

---

## What "Active Parameters" Means (26B A4B)

The "A" in 26B A4B stands for **Active** — the number of parameters used per inference pass. The full model has 25.2B total parameters across 128 experts, but a router selects only a subset of experts per token, resulting in ~3.8B parameters being active at any given time.

This is different from E2B/E4B's "effective" label: with A4B, the inactive parameters are still physically loaded in memory — they're just not computed for that token. 
You get:
- **the speed of a ~4B model** with 
- **the knowledge capacity of a 25B model**, but 
- **pay the memory cost of the full 25B**

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

> CPU core counts below are practical estimates based on llama.cpp/Ollama inference patterns, not official Google specs.

| Model | RAM (Q4) | vCPUs (min) | vCPUs (recommended) | GPU needed? |
|-------|----------|-------------|---------------------|-------------|
| E2B | 4 GB | 2 | 4 | No |
| E4B | 6 GB | 4 | 8 | No |
| 26B A4B | 16–18 GB | 8 | 16 | Strongly recommended |
| 31B | 17–20 GB | 16 | 32 | Yes |

More cores primarily help with memory bandwidth when loading weights — beyond 16 cores the returns diminish for inference on a single model instance. For the 26B/31B, a GPU matters far more than core count.


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
- [Gemma 4 for Edge Deployment: E2B and E4B — MindStudio](https://www.mindstudio.ai/blog/gemma-4-edge-deployment-e2b-e4b-models)
- [Gemma 4 model card — Google AI for Developers](https://ai.google.dev/gemma/docs/core/model_card_4)
- [google/gemma-4-E2B — Hugging Face](https://huggingface.co/google/gemma-4-E2B)
- [google/gemma-4-E4B — Hugging Face](https://huggingface.co/google/gemma-4-E4B)
- [I Ran Gemma 4 Models on 48GB GPU — DEV Community](https://dev.to/gaurav_vij137/i-ran-googles-latest-gemma-4-models-on-48gb-gpu-heres-what-actually-happened-5d3d)
- [Benchmarking Gemma 4 26B and 31B Locally — n1n.ai](https://explore.n1n.ai/blog/benchmarking-google-gemma-4-26b-31b-locally-2026-04-06)
- [Gemma 4 — How to Run Locally — Unsloth](https://unsloth.ai/docs/models/gemma-4)
