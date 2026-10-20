# gpt-oss-120b on vLLM (one Spark)

| | |
|---|---|
| Model | [`openai/gpt-oss-120b`](https://huggingface.co/openai/gpt-oss-120b), MXFP4, 116.8B total / 5.1B active |
| Memory | Weights ~59 GiB as GGUF ([llama.cpp bench](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)). The full HF download is ~196 GB on disk ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md)). About 117 GB of the 128 GB is usable ([forum](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126)); the OS sees ~121 GiB ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes)). Within `--gpu-memory-utilization`, whatever the weights leave goes to KV cache |
| Expected speed | **Path B: 60.7 tok/s with 1 user, 116–153 tok/s in total with 8 users, TTFT 131 ms** ([ss121, 2026-08-10](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766)). The spark-vllm-docker changelog states "60 t/s on a single Spark" ([2026-01-29](https://github.com/eugr/spark-vllm-docker)). Path A: no published number for the current NGC image |
| Recipe source | A: legacy NVIDIA playbook [`nvidia/vllm/README.md`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md) (Last Updated 06/12/2026). The current [build.nvidia.com/spark/vllm](https://build.nvidia.com/spark/vllm) page (09/14/2026) is built around NVIDIA Sync and points to [recipes.vllm.ai](https://recipes.vllm.ai/browse?panel=open&hw=dgx_spark_gb10) and spark-vllm-docker; the NGC commands and the tested-model matrix are only in the GitHub README. B: community [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker), recipe `recipes/openai-gpt-oss-120b.yaml` |
| Status | A: **official** (gpt-oss-120b is in the legacy playbook's tested MXFP4 matrix). B: **community-measured** |

## Path A: official NGC container (simplest)

```bash
export HF_TOKEN="your_huggingface_token"
export LATEST_VLLM_VERSION=26.05.post1-py3     # example tag from the playbook; 26.09-py3 is the newest stable tag on nvcr.io as of 2026-10-20
export HF_MODEL_HANDLE=openai/gpt-oss-120b

docker pull nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION}

docker run -it --gpus all -p 8000:8000 \
nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION} \
vllm serve ${HF_MODEL_HANDLE}
```

To keep the download between runs, add `-v ~/.cache/huggingface:/root/.cache/huggingface -e HF_TOKEN`. *(modified from the playbook command; untested)*

## Path B: MXFP4 CUTLASS build (fastest measured)

This path needs a local image build. There is no prebuilt MXFP4 image, and field-notes timed the build at about 26 minutes on 20 cores.

```bash
git clone https://github.com/eugr/spark-vllm-docker.git
cd spark-vllm-docker
curl -LsSf https://astral.sh/uv/install.sh | sh     # hf-download.sh needs uvx

./build-and-copy.sh --exp-mxfp4 -j 20
./hf-download.sh openai/gpt-oss-120b
./run-recipe.sh openai-gpt-oss-120b --solo
```

The recipe runs this command (copied from the repo's YAML recipe, with `VLLM_USE_FLASHINFER_MOE_MXFP4_MXFP8=1` set in its env):

```bash
vllm serve openai/gpt-oss-120b \
    --tool-call-parser openai \
    --reasoning-parser openai_gptoss \
    --enable-auto-tool-choice \
    --gpu-memory-utilization 0.8 \
    --enable-prefix-caching \
    --load-format fastsafetensors \
    --quantization mxfp4 \
    --mxfp4-backend CUTLASS \
    --mxfp4-layers moe,qkv,o,lm_head \
    --attention-backend FLASHINFER \
    --kv-cache-dtype fp8 \
    --max-num-batched-tokens 8192 \
    --host 0.0.0.0 \
    --port 8000
```

Check the log before you trust a number. You should see `[MXFP4] Using backend: CUTLASS` ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md)).

## Test

```bash
curl -s http://localhost:8000/v1/models
curl http://localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{
  "model": "openai/gpt-oss-120b",
  "messages": [{"role": "user", "content": "Hello"}],
  "max_tokens": 512
}'
```

`max_tokens` must leave room for reasoning. Otherwise thinking models return an empty `content` with `finish_reason: length` (troubleshooting in the legacy vLLM playbook).

## Notes

- Do not pass `--enforce-eager`. It disables CUDA graphs; field-notes measured NVFP4 at 23 tok/s with it and 67 tok/s without it.
- A cold start takes about 4 minutes (load, `torch.compile`, graph capture). Throw away the first request before you measure anything (field-notes).
- Two Sparks: a patched vLLM 0.17.0 with TP=2 reached 81.5 tok/s ([forum, 2026-03-11](https://forums.developer.nvidia.com/t/vllm-0-17-0-mxfp4-patches-for-dgx-spark-qwen3-5-35b-a3b-70-tok-s-gpt-oss-120b-80-tok-s-tp-2/362824)). See [cluster/two-sparks.md](../cluster/two-sparks.md).
