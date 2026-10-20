# gpt-oss-20b on vLLM

| | |
|---|---|
| Model | [`openai/gpt-oss-20b`](https://huggingface.co/openai/gpt-oss-20b), MXFP4, 20.9B total / 3.6B active |
| Memory | Weights ~11 GiB (GGUF size in the [llama.cpp bench](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)). Most of the vLLM reservation goes to KV cache |
| Expected speed | ASUS GX10 (same GB10 chip), vLLM: **~90 tok/s with 1 user, ~1,600 tok/s in total with 64 users** (rounded by the author; quantisation not stated) ([StorageReview, 2026-03-03](https://www.storagereview.com/review/asus-ascent-gx10-review)). For comparison, llama.cpp: 83.4 tok/s with 1 user ([ggerganov, 2026-02-05](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)) |
| Recipe source | NVIDIA playbook [`vllm`](https://build.nvidia.com/spark/vllm). This is the playbook's own example model |
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

- For one interactive user, llama.cpp is at least as fast and starts in seconds: [llama.cpp/gpt-oss.md](../llama.cpp/gpt-oss.md).
- Use vLLM when several clients send requests at the same time. Batching is where it wins.
