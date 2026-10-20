# gpt-oss-120b and gpt-oss-20b on SGLang

| | |
|---|---|
| Models | [`openai/gpt-oss-120b`](https://huggingface.co/openai/gpt-oss-120b) · [`openai/gpt-oss-20b`](https://huggingface.co/openai/gpt-oss-20b) (MXFP4) |
| Image | `lmsysorg/sglang:latest-cu130` (CUDA 13; multi-arch with arm64 on Docker Hub, checked 2026-10-20) |
| Expected speed, ISL/OSL 2048 | **120b: 50.5 tok/s with 1 user, 117.1 in total with 8, 203.1 with 32** · **20b: 69.6 / 219.3 / 521.9** (image `lmsysorg/sglang:spark`, [LMSYS, 2025-10-13](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark)). These are launch-day numbers, and the image has changed since |
| Recipe source | NVIDIA playbook [`sglang`](https://build.nvidia.com/spark/sglang), Last Updated 07/31/2026 |
| Status | **official.** The playbook's example is Qwen3-8B; gpt-oss-20b and -120b are on its tested-model list |

## Run

```bash
export HF_TOKEN=""
export MODEL_HANDLE="openai/gpt-oss-120b"
export MAX_MODEL_LEN=8192
export SGLANG_IMAGE="lmsysorg/sglang:latest-cu130"   # CUDA 13.0 — required for Blackwell

docker pull "$SGLANG_IMAGE"
docker run -d \
  --name sglang-server \
  --gpus all \
  --ipc host \
  --cap-add SYS_NICE \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  -p 30000:30000 \
  -e HF_TOKEN="$HF_TOKEN" \
  -v "$HOME/.cache/huggingface/hub:/root/.cache/huggingface/hub" \
  "$SGLANG_IMAGE" \
  sglang serve --model-path "$MODEL_HANDLE" \
    --host 0.0.0.0 \
    --port 30000 \
    --context-length $MAX_MODEL_LEN \
    --mem-fraction-static 0.85 \
    --attention-backend flashinfer \
    --enable-cache-report \
    --trust-remote-code
```

- For NVFP4 checkpoints, add `--quantization modelopt_fp4`.
- To see prefix-cache hits: `docker logs sglang-server 2>&1 | grep "cached-token"`.

## Other models on the playbook's tested list

Llama-3.1-8B (FP8, NVFP4), Llama-3.3-70B (NVFP4), Qwen3-8B and Qwen3-14B (FP8, NVFP4), Qwen3-32B (NVFP4), Phi-4, Nemotron-3-Nano-Omni.

## Speculative decoding

At launch, LMSYS showed EAGLE3 on Llama 3.1 8B giving up to ~2x (draft `jamesliu1/sglang-EAGLE3-Llama-3.1-Instruct-8B`, steps 5, topk 8, 32 draft tokens, mem fraction 0.6, cuda-graph max bs 2) ([LMSYS](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark)).
