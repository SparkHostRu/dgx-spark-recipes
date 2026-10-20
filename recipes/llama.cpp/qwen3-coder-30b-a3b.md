# Qwen3-Coder-30B-A3B on llama.cpp

| | |
|---|---|
| Model | [`ggml-org/Qwen3-Coder-30B-A3B-Instruct-Q8_0-GGUF`](https://huggingface.co/ggml-org/Qwen3-Coder-30B-A3B-Instruct-Q8_0-GGUF), 30.25 GiB, 30.5B total / 3B active |
| Expected speed, 1 user | **61.1 tok/s decode, 2,987 tok/s prefill** at empty context; 30.2 / 1,348 at 32k depth ([ggerganov, build b7941, 2026-02-05](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)) |
| Expected speed, many users | 120.4 tok/s in total with 8 users, 216.8 with 32 (`llama-batched-bench`, 4,096-token prompt, same source) |
| Recipe source | Model and numbers: llama.cpp upstream benches. Server flags: NVIDIA playbook [`llama-cpp`](https://build.nvidia.com/spark/llama-cpp) |
| Status | Numbers: **community-measured** (`llama-bench`). Server command: **untested, from the llama-cpp playbook** (the playbook's flags with this model swapped in) |

Build first: [build.md](build.md).

## Run

```bash
./build/bin/llama-server \
  -hf ggml-org/Qwen3-Coder-30B-A3B-Instruct-Q8_0-GGUF \
  --host 0.0.0.0 \
  --port 30000 \
  --ctx-size 100000 \
  --jinja
```

The playbook recommends `--ctx-size` of at least 32,768 for agent and coding work, and 100,000+ if memory allows.

## Notes

- Q8_0 is the quant that was measured. A smaller quant, such as Unsloth's [`unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF`](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF), reads fewer bytes per token and should decode faster. We have no Spark measurement for it.
- Decode slows down as context grows: 61 tok/s at empty context, 30 tok/s at 32k. Plan agent sessions with that in mind.
