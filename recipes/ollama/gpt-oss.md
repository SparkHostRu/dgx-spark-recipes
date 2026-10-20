# gpt-oss-120b and gpt-oss-20b on Ollama

| | |
|---|---|
| Models | `gpt-oss:120b` (116.8B total / 5.1B active, MXFP4) · `gpt-oss:20b` (20.9B / 3.6B, MXFP4) |
| Download size | 65.4 GB (`120b`) · 13.8 GB (`20b`). Sum of layer sizes in the [Ollama registry](https://ollama.com/library/gpt-oss) manifests, checked 2026-10-20 |
| Expected speed, 1 user | **120b: 42.1 tok/s** ([ss121, 2026-08-10](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766)), 41.1 tok/s ([Ollama blog, v0.12.6, 2025-10-23](https://ollama.com/blog/nvidia-spark-performance)) · **20b: 58.3 tok/s** (Ollama blog) |
| Expected speed, 8 users | 120b: 42.5 tok/s in total (ss121). With the default `OLLAMA_NUM_PARALLEL=1`, Ollama serialises requests, so 8 users share one user's speed |
| Recipe source | Install: NVIDIA playbook [`ollama`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/ollama/README.md) (Last Updated 10/12/2025). Tuning: [dgx-spark-field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/serving-stacks.md) |
| Status | Install: **official**. Model tags `gpt-oss:120b` / `gpt-oss:20b`: **untested, from ollama playbook** (the playbook pulls `qwen2.5:32b`; the speeds above come from Ollama's blog and ss121, not from this exact command). Env tuning: **community-measured** |

## Run

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull gpt-oss:120b        # or gpt-oss:20b
ollama run gpt-oss:120b
```

API check, same request format as the playbook:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gpt-oss:120b",
  "messages": [{"role": "user", "content": "Write me a haiku about GPUs and AI."}],
  "stream": false
}'
```

## Tuning

The field-notes measurements ran Ollama with these server variables:

```bash
OLLAMA_FLASH_ATTENTION=1
OLLAMA_KV_CACHE_TYPE=q8_0
OLLAMA_CONTEXT_LENGTH=16384
OLLAMA_KEEP_ALIVE=30m
OLLAMA_MAX_LOADED_MODELS=3
```

The install script runs Ollama as a systemd service. To set the variables, add `Environment="NAME=value"` lines with `sudo systemctl edit ollama`, then restart the service ([Ollama FAQ](https://docs.ollama.com/faq)). *untested, from Ollama FAQ.*

## When to switch engines

- **Several users or an API:** use vLLM. On the same harness, vLLM was faster for gpt-oss-120b both for one user (60.7 vs 42.1 tok/s) and for 8 users. See [vllm/gpt-oss-120b.md](../vllm/gpt-oss-120b.md).
- **Agents and tool calling:** field-notes report that tool calling with `gpt-oss:120b` fails under Ollama and works under vLLM with `--tool-call-parser openai --reasoning-parser openai_gptoss --enable-auto-tool-choice` ([troubleshooting](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/troubleshooting.md)).
- **Remote access:** the NVIDIA playbook exposes port 11434 through NVIDIA Sync (custom app with an SSH tunnel).
