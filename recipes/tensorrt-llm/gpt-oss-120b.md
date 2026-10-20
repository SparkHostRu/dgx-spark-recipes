# gpt-oss-120b on TensorRT-LLM (with EAGLE-3)

| | |
|---|---|
| Model | [`openai/gpt-oss-120b`](https://huggingface.co/openai/gpt-oss-120b) (MXFP4). EAGLE-3 draft head: [`nvidia/gpt-oss-120b-Eagle3-long-context`](https://huggingface.co/nvidia/gpt-oss-120b-Eagle3-long-context) |
| Image | `nvcr.io/nvidia/tensorrt-llm/release:1.3.0rc13` (plain serve) / `1.3.0rc12` (speculative decoding). Both have arm64 manifests (checked 2026-10-20) |
| Memory | NVIDIA states one Spark is enough for gpt-oss-120b with EAGLE-3 ([playbook `speculative-decoding`](https://build.nvidia.com/spark/speculative-decoding)) |
| Expected speed | **No published TensorRT-LLM tok/s for gpt-oss-120b on Spark found.** For comparison, vLLM reached 60.7 tok/s with 1 user ([vllm/gpt-oss-120b.md](../vllm/gpt-oss-120b.md)) |
| Recipe source | NVIDIA playbooks [`trt-llm`](https://build.nvidia.com/spark/trt-llm) (Last Updated 04/28/2026) and [`speculative-decoding`](https://build.nvidia.com/spark/speculative-decoding) (04/20/2026) |
| Status | **official** (gpt-oss-120b is in the trt-llm matrix; EAGLE-3 is the speculative-decoding playbook's own example) |

## Plain serve

The playbook's example model is gpt-oss-20b. Here it is switched to 120b, which is also on the matrix.

```bash
export HF_TOKEN=<your-huggingface-token>
export DOCKER_IMAGE="nvcr.io/nvidia/tensorrt-llm/release:1.3.0rc13"
export MODEL_HANDLE="openai/gpt-oss-120b"

docker run --name trtllm_llm_server --rm -it --gpus all --ipc host --network host \
  -e HF_TOKEN=$HF_TOKEN \
  -e MODEL_HANDLE="$MODEL_HANDLE" \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  $DOCKER_IMAGE \
  bash -c '
    export TIKTOKEN_ENCODINGS_BASE="/tmp/harmony-reqs" && \
    mkdir -p $TIKTOKEN_ENCODINGS_BASE && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken && \
    hf download $MODEL_HANDLE && \
    cat > /tmp/extra-llm-api-config.yml <<EOF
print_iter_log: false
kv_cache_config:
  dtype: "auto"
  free_gpu_memory_fraction: 0.9
cuda_graph_config:
  enable_padding: true
disable_overlap_scheduler: true
EOF
    trtllm-serve "$MODEL_HANDLE" \
      --max_batch_size 64 \
      --trust_remote_code \
      --port 8355 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml
  '
```

gpt-oss needs the tiktoken encodings (`TIKTOKEN_ENCODINGS_BASE`), as shown above.

## EAGLE-3 speculative decoding

```bash
docker run \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  nvcr.io/nvidia/tensorrt-llm/release:1.3.0rc12 \
  bash -c '
    hf download openai/gpt-oss-120b && \
    hf download nvidia/gpt-oss-120b-Eagle3-long-context \
        --local-dir /opt/gpt-oss-120b-Eagle3/ && \
    cat > /tmp/extra-llm-api-config.yml <<EOF
enable_attention_dp: false
disable_overlap_scheduler: false
enable_autotuner: false
cuda_graph_config:
    max_batch_size: 1
speculative_config:
    decoding_type: Eagle
    max_draft_len: 5
    speculative_model_dir: /opt/gpt-oss-120b-Eagle3/

kv_cache_config:
    free_gpu_memory_fraction: 0.9
    enable_block_reuse: false
EOF
    export TIKTOKEN_ENCODINGS_BASE="/tmp/harmony-reqs" && \
    mkdir -p $TIKTOKEN_ENCODINGS_BASE && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken
    trtllm-serve openai/gpt-oss-120b \
      --backend pytorch --tp_size 1 \
      --max_batch_size 1 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml'
```

`max_batch_size: 1` tunes this config for a single user.

## Troubleshooting (from the playbook)

- OOM while loading weights: `export TRT_LLM_DISABLE_LOAD_WEIGHTS_IN_PARALLEL=1`.
- To put Open WebUI in front of it, set `OPENAI_API_BASE_URL=http://localhost:8355/v1`. See [open-webui](../open-webui/open-webui.md).
