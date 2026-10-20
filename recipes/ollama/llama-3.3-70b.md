# Llama 3.3 70B on Ollama

| | |
|---|---|
| Model | `llama3.3:70b` (dense 70B, Q4_K_M) |
| Download size | 42.5 GB ([Ollama registry](https://ollama.com/library/llama3.3) manifest, checked 2026-10-20) |
| Expected speed, 1 user | **~5 tok/s** ([dgx-spark-field-notes, hardware model](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/hardware-model.md), 2026-08). Llama **3.1** 70B Q4_K_M, same parameter count: 4.42 tok/s ([Ollama blog, 2025-10-23](https://ollama.com/blog/nvidia-spark-performance)), 4.58 tok/s ([LMSYS, 2025-10-13](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark)) |
| Recipe source | Install: NVIDIA playbook [`ollama`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/ollama/README.md). Tag: Ollama library |
| Status | Install: **official**. Tag: **community-measured** (approximate figure from field-notes) |

## Run

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.3:70b
ollama run llama3.3:70b
```

## Why it is slow

A dense model reads all of its weights for every token. Decode on Spark is limited by memory bandwidth (273 GB/s, [NVIDIA spec](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)). A 4-bit 70B model has roughly 35 GB of weights, so a rough upper bound is 273 / 35 ≈ 7.8 tok/s. This is arithmetic, not a measurement.

MoE models only read their active experts. On the same machine `gpt-oss:120b` ran at 42.1 tok/s against ~5 tok/s for `llama3.3:70b` (field-notes). If you do not specifically need Llama 3.3, start with [gpt-oss](gpt-oss.md) or [Qwen3-30B-A3B](qwen3.md).

If you do need Llama 3.3 70B:
- NVFP4 on vLLM or TensorRT-LLM is officially supported: [vllm/llama-3.3-70b.md](../vllm/llama-3.3-70b.md), [tensorrt-llm/llama-3.3-70b.md](../tensorrt-llm/llama-3.3-70b.md).
- TensorRT-LLM with an 8B draft model (speculative decoding) is NVIDIA's own recipe for this model.
