# Llama 3.3 70B on TensorRT-LLM (with an 8B draft model)

| | |
|---|---|
| Models | Target: `nvidia/Llama-3.3-70B-Instruct-FP4` (Hugging Face redirects it to [`nvidia/Llama-3.3-70B-Instruct-NVFP4`](https://huggingface.co/nvidia/Llama-3.3-70B-Instruct-NVFP4)). Draft: `nvidia/Llama-3.1-8B-Instruct-FP4` (redirects to `-NVFP4`) |
| Image | `nvcr.io/nvidia/tensorrt-llm/release:1.3.0rc12` (arm64 manifest present, checked 2026-10-20) |
| Memory | NVIDIA states one Spark is enough for Llama-3.3-70B with draft-target ([playbook `speculative-decoding`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/speculative-decoding/README.md)) |
| Expected speed | **No published tok/s for this recipe on Spark found.** For reference, dense 70B without speculation: Ollama ~5 tok/s ([ollama/llama-3.3-70b.md](../ollama/llama-3.3-70b.md)) |
| Recipe source | NVIDIA playbook [`speculative-decoding`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/speculative-decoding/README.md), Option 2 "Draft Target", Last Updated 04/20/2026 |
| Status | **official** |

## Run

```bash
export HF_TOKEN=<your_huggingface_token>

docker run \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host nvcr.io/nvidia/tensorrt-llm/release:1.3.0rc12 \
  bash -c "
    hf download nvidia/Llama-3.3-70B-Instruct-FP4 && \
    hf download nvidia/Llama-3.1-8B-Instruct-FP4 \
    --local-dir /opt/Llama-3.1-8B-Instruct-FP4/ && \
    cat <<EOF > extra-llm-api-config.yml
print_iter_log: false
disable_overlap_scheduler: true
speculative_config:
  decoding_type: DraftTarget
  max_draft_len: 4
  speculative_model_dir: /opt/Llama-3.1-8B-Instruct-FP4/
kv_cache_config:
  enable_block_reuse: false
EOF
    trtllm-serve nvidia/Llama-3.3-70B-Instruct-FP4 \
      --backend pytorch --tp_size 1 \
      --max_batch_size 1 \
      --kv_cache_free_gpu_memory_fraction 0.9 \
      --extra_llm_api_options ./extra-llm-api-config.yml
  "
```

Test:

```bash
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Llama-3.3-70B-Instruct-FP4",
    "prompt": "Explain the benefits of speculative decoding:",
    "max_tokens": 150,
    "temperature": 0.7
  }'
```

The playbook suggests trying `max_draft_len` values of 1, 2, 3, 4 and 8 and watching the acceptance rate. If you measure tok/s, please [share the result](../../CONTRIBUTING.md). This recipe has no public number yet.

## Without speculation

Llama-3.3-70B NVFP4 is also on the plain `trt-llm` playbook matrix. Use the command template from [gpt-oss-120b.md](gpt-oss-120b.md#plain-serve) with `MODEL_HANDLE=nvidia/Llama-3.3-70B-Instruct-NVFP4` and drop the tiktoken lines, which only gpt-oss needs. *(untested, from the trt-llm playbook)*
