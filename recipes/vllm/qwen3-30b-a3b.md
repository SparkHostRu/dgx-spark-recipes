# Qwen3-30B-A3B and Qwen3-Coder-30B-A3B on vLLM

| | |
|---|---|
| Models | Qwen3-30B-A3B as NVFP4 ([`nvidia/Qwen3-30B-A3B-NVFP4`](https://huggingface.co/nvidia/Qwen3-30B-A3B-NVFP4) or [`RedHatAI/Qwen3-30B-A3B-NVFP4`](https://huggingface.co/RedHatAI/Qwen3-30B-A3B-NVFP4)) · Qwen3-Coder-30B-A3B as FP8 ([`Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8`](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8)) |
| Memory | Fits comfortably on one Spark. Field-notes reported 76.55 GiB of KV cache for a 17.7 GiB model at `--gpu-memory-utilization 0.8` ([troubleshooting](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/troubleshooting.md)) |
| Expected speed, Qwen3-30B-A3B NVFP4 | **77.1 tok/s with 1 user, 313.4 tok/s in total with 8 users** (image `eugr/spark-vllm:latest`) ([ss121, 2026-08-10](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766)) |
| Qwen3-Coder-30B-A3B FP8: decode | No published vLLM decode figure on Spark found |
| Qwen3-Coder-30B-A3B FP8: total throughput (not decode) | Total input + output tokens/s, Equal ISL/OSL profile: 46.5 at 1 user, 482.6 at 64 users ([StorageReview, 2025-10-14](https://storagereview.com/review/nvidia-dgx-spark-review-the-ai-appliance-bringing-datacenter-capabilities-to-desktops)); ASUS GX10: ~100 / ~1,230 (rounded, [StorageReview, 2026-03-03](https://www.storagereview.com/review/asus-ascent-gx10-review)). Do not compare with decode numbers |
| Recipe source | Command layout: [dgx-spark-field-notes, vllm-on-spark](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md) (*modified from*: model name filled in, `PYTORCH_CUDA_ALLOC_CONF` added as that page recommends). Images: community `eugr/spark-vllm` (arm64 only) or NVIDIA NGC `nvcr.io/nvidia/vllm` |
| Status | **untested, from dgx-spark-field-notes.** The measurement above does not name the exact NVFP4 checkpoint, so the checkpoint below is our choice. The Coder FP8 command is our adaptation; StorageReview did not publish theirs |

## Qwen3-30B-A3B, NVFP4

```bash
docker run --gpus all --ipc=host --network host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -e PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True \
  eugr/spark-vllm:latest \
  vllm serve nvidia/Qwen3-30B-A3B-NVFP4 \
    --host 0.0.0.0 --port 8000 \
    --gpu-memory-utilization 0.8 \
    --enable-prefix-caching \
    --load-format fastsafetensors \
    --kv-cache-dtype fp8
```

Check the log for `Using FlashInferCutlassNvFp4LinearKernel`. Without the sm_121 kernels, NVFP4 falls back to emulation and drops to ~1.1 tok/s (ss121).

## Qwen3-Coder-30B-A3B, FP8 (NGC image)

```bash
docker run -it --gpus all -p 8000:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  nvcr.io/nvidia/vllm:26.05.post1-py3 \
  vllm serve Qwen/Qwen3-Coder-30B-A3B-Instruct-FP8 \
    --gpu-memory-utilization 0.8 \
    --enable-auto-tool-choice --tool-call-parser qwen3_coder
```

`--tool-call-parser qwen3_coder` is the parser the spark-vllm-docker recipes use for Qwen3-Coder-Next.

## When to use something else

- **One user:** Ollama `qwen3:30b-a3b` was 11% faster on the same harness (85.2 tok/s). See [ollama/qwen3.md](../ollama/qwen3.md).
- **Qwen3-Coder-30B-A3B for one user:** llama.cpp Q8_0 measured 61.1 tok/s ([llama.cpp/qwen3-coder-30b-a3b.md](../llama.cpp/qwen3-coder-30b-a3b.md)).
- **Officially tested alternative:** TensorRT-LLM lists Qwen3-30B-A3B NVFP4 in its Spark matrix ([playbook `trt-llm`](https://build.nvidia.com/spark/trt-llm)).
