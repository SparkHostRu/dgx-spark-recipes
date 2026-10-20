# Qwen3-Coder-Next on vLLM

| | |
|---|---|
| Model | Qwen3-Coder-Next (80B-A3B class MoE). One Spark: [`Intel/Qwen3-Coder-Next-int4-AutoRound`](https://huggingface.co/Intel/Qwen3-Coder-Next-int4-AutoRound). Two Sparks: [`Qwen/Qwen3-Coder-Next-FP8`](https://huggingface.co/Qwen/Qwen3-Coder-Next-FP8) |
| Memory | INT4 recipe: `gpu_memory_utilization 0.8`, `max_model_len 262144`, `solo_only: true`. The FP8 recipe defaults to TP=2 (two Sparks) |
| Expected speed | INT4 AutoRound, 1 Spark: **74.8 tok/s with 1 user, 289.0 tok/s in total with 8 users, TTFT 104 ms, ~3,785 tok/s prefill on a 10k prompt** ([ss121, 2026-08-10](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766); [results.jsonl](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/benchmarks/results.jsonl)) |
| Recipe source | Community [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker): `recipes/qwen3-coder-next-int4-autoround.yaml`, `recipes/qwen3-coder-next-fp8.yaml` |
| Status | INT4, one Spark: **community-measured**. FP8, two Sparks: **untested, from spark-vllm-docker** (no published result) |

## One Spark, INT4 AutoRound

```bash
git clone --depth 1 https://github.com/eugr/spark-vllm-docker.git
cd spark-vllm-docker

docker pull eugr/spark-vllm:latest
docker tag eugr/spark-vllm:latest vllm-node:latest     # the scripts expect this local tag

./run-recipe.sh qwen3-coder-next-int4-autoround --solo --dry-run   # prints the exact command
./run-recipe.sh qwen3-coder-next-int4-autoround --solo
```

The recipe applies the mod `mods/fix-qwen3-next-autoround`, sets `VLLM_MARLIN_USE_ATOMIC_ADD=1`, and runs:

```bash
vllm serve Intel/Qwen3-Coder-Next-int4-AutoRound \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder \
  --gpu-memory-utilization 0.8 \
  --host 0.0.0.0 \
  --port 8000 \
  --load-format instanttensor \
  --enable-prefix-caching \
  --max-model-len 262144
```

## Two Sparks, FP8

The FP8 recipe defaults to `tensor_parallel: 2` with `--distributed-executor-backend ray`. Set up the cluster first ([cluster/two-sparks.md](../cluster/two-sparks.md)), then:

```bash
./run-recipe.sh qwen3-coder-next-fp8 --setup
```

## Notes

- Mods in that repo drift against vLLM nightlies. Field-notes found one mod that no longer applied. Run `--dry-run` first.
- With Ollama (`qwen3-coder-next`, q4_K_M) the same harness gave 59.2 tok/s: [ollama/qwen3.md](../ollama/qwen3.md).
