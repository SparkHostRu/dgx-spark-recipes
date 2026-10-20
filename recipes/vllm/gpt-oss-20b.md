# gpt-oss-20b on vLLM

| | |
|---|---|
| Model | [`openai/gpt-oss-20b`](https://huggingface.co/openai/gpt-oss-20b), MXFP4, 20.9B total / 3.6B active |
| Memory | Weights ~11 GiB (GGUF size in the [llama.cpp bench](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)). Most of the vLLM reservation goes to KV cache |
| Expected speed (decode) | No published single-user decode figure for vLLM on Spark found. For reference, llama.cpp decodes at 83.4 tok/s with 1 user ([ggerganov, 2026-02-05](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)) |
| Total throughput (not decode) | ASUS GX10 (same GB10 chip), vLLM, Equal ISL/OSL profile: ~90 tok/s at 1 user, ~1,600 tok/s at 64 users. These are **total input + output tokens/s**, rounded by the author, quantisation not stated ([StorageReview, 2026-03-03](https://www.storagereview.com/review/asus-ascent-gx10-review)). Do not compare them with decode numbers |
| Recipe source | legacy NVIDIA playbook [`nvidia/vllm/README.md`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md) (Last Updated 06/12/2026). The current [build.nvidia.com/spark/vllm](https://build.nvidia.com/spark/vllm) page (09/14/2026) is built around NVIDIA Sync and points to [recipes.vllm.ai](https://recipes.vllm.ai/browse?panel=open&hw=dgx_spark_gb10) and spark-vllm-docker; the NGC commands and the tested-model matrix are only in the GitHub README. gpt-oss-20b is the playbook's own example model |
| Status | **official** |

## Run

```bash
export HF_TOKEN="your_huggingface_token"
export LATEST_VLLM_VERSION=26.05.post1-py3
export HF_MODEL_HANDLE=openai/gpt-oss-20b

docker pull nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION}

docker run -it --gpus all -p 8000:8000 \
nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION} \
vllm serve ${HF_MODEL_HANDLE}
```

The same setup as a compose file with Open WebUI: [compose/vllm-openwebui](../../compose/vllm-openwebui/).

## Notes

- For one interactive user, llama.cpp has a published decode figure (83.4 tok/s) and starts in seconds: [llama.cpp/gpt-oss.md](../llama.cpp/gpt-oss.md).
- Use vLLM when several clients send requests at the same time. Batching is where it wins.
