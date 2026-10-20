# Qwen3 family on Ollama: Qwen3-30B-A3B, Qwen3-Coder 30B, Qwen3-Coder-Next, Qwen3-Next

| | |
|---|---|
| Models (Ollama tags) | `qwen3:30b-a3b` · `qwen3-coder:30b` · `qwen3-coder-next` · `qwen3-next:80b` |
| Download size | 18.6 GB · 18.6 GB · 51.7 GB · 50.4 GB ([Ollama registry](https://ollama.com/library) manifests, checked 2026-10-20) |
| Expected speed, 1 user | **`qwen3:30b-a3b`: 85.2 tok/s** · **`qwen3-coder-next` (q4_K_M): 59.2 tok/s** ([ss121, 2026-08-10](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766)). `qwen3-coder:30b` and `qwen3-next:80b`: no published Ollama number on Spark found |
| Expected speed, 8 users | `qwen3:30b-a3b` 87.5 · `qwen3-coder-next` 60.4 tok/s in total (default `OLLAMA_NUM_PARALLEL=1` serialises requests) |
| Recipe source | Install: NVIDIA playbook [`ollama`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/ollama/README.md). Tags: Ollama library |
| Status | `qwen3:30b-a3b`, `qwen3-coder-next`: **community-measured**. `qwen3-coder:30b`, `qwen3-next:80b`: **untested, from Ollama library** (tags exist; no Spark measurement) |

## Run

```bash
curl -fsSL https://ollama.com/install.sh | sh

ollama pull qwen3:30b-a3b           # general use, fastest single-user Qwen3 here
ollama pull qwen3-coder:30b         # coding, same 30B-A3B size class
ollama pull qwen3-coder-next        # coding, 80B-A3B class, ~52 GB
ollama run qwen3:30b-a3b
```

The [gpt-oss recipe](gpt-oss.md#tuning) lists the server variables used in the measurements.

## Notes

- **Qwen3-30B-A3B is the one model where Ollama beat vLLM for a single user** on the same harness: 85.2 vs 77.1 tok/s (ss121). With 8 parallel requests vLLM delivered 313.4 tok/s in total and Ollama 87.5. See [vllm/qwen3-30b-a3b.md](../vllm/qwen3-30b-a3b.md).
- **Qwen3-Coder-Next:** vLLM with an INT4 AutoRound checkpoint was faster, at 74.8 vs 59.2 tok/s. See [vllm/qwen3-coder-next.md](../vllm/qwen3-coder-next.md).
- **Prefill:** at a ~10k-token prompt the same author measured 1,439 tok/s for `qwen3-coder-next` under Ollama and 3,785 tok/s under vLLM ([results.jsonl](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/benchmarks/results.jsonl)). For agents with long prompts this matters more than decode speed.
- **For a Qwen3-Coder-30B-A3B measurement on Spark**, see llama.cpp: 61.1 tok/s at Q8_0 ([llama.cpp/qwen3-coder-30b-a3b.md](../llama.cpp/qwen3-coder-30b-a3b.md)).
