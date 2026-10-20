# Llama 3.3 70B on vLLM

| | |
|---|---|
| Model | One Spark: [`nvidia/Llama-3.3-70B-Instruct-NVFP4`](https://huggingface.co/nvidia/Llama-3.3-70B-Instruct-NVFP4). Two Sparks: [`meta-llama/Llama-3.3-70B-Instruct`](https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct) (gated, needs `HF_TOKEN`) |
| Memory | NVFP4 weights are about a quarter of BF16 (~35 GB for 70B, our arithmetic). BF16 (~140 GB) needs two Sparks |
| Expected speed | **No published vLLM measurement on Spark found.** Decode is bandwidth-bound: 273 GB/s ÷ ~35 GB ≈ 7.8 tok/s upper bound for one 4-bit request (arithmetic, not a measurement). Closest measured: Ollama `llama3.3:70b` ~5 tok/s ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/hardware-model.md)); Llama 3.1 70B FP8 on SGLang 2.66 tok/s with 1 user and 20.2 tok/s in total with 8 users ([LMSYS, 2025-10-13](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark)) |
| Recipe source | NVIDIA playbook [`vllm`](https://build.nvidia.com/spark/vllm): the single-Spark NGC command, and the "Run on two Sparks" tab |
| Status | **official.** Both models are in the playbook (NVFP4 in the tested matrix; BF16 TP=2 is the two-Spark example) |

## One Spark, NVFP4

```bash
export HF_TOKEN="your_huggingface_token"
export LATEST_VLLM_VERSION=26.05.post1-py3
export HF_MODEL_HANDLE=nvidia/Llama-3.3-70B-Instruct-NVFP4

docker pull nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION}

docker run -it --gpus all -p 8000:8000 \
nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION} \
vllm serve ${HF_MODEL_HANDLE}
```

## Two Sparks, BF16, TP=2

Set up Ray across both nodes as in [cluster/two-sparks.md](../cluster/two-sparks.md#3a-vllm-ray--ngc-official), then on the head node:

```bash
docker exec -it $VLLM_CONTAINER /bin/bash -c '
  vllm serve meta-llama/Llama-3.3-70B-Instruct \
    --tensor-parallel-size 2 --max-model-len 2048 \
    --gpu-memory-utilization 0.8 --distributed-executor-backend ray'
```

## Notes

- If you need more speed from this model, NVIDIA's own answer is speculative decoding with an 8B draft model on TensorRT-LLM: [tensorrt-llm/llama-3.3-70b.md](../tensorrt-llm/llama-3.3-70b.md).
- With NVFP4, check that vLLM picked a real FP4 kernel and is not emulating. Field-notes and the NVIDIA forum report ~1.1 tok/s in emulation ([forum, 2026-08-10](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766)).
