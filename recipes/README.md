# Recipes

One file per engine and model. Every file starts with the same header block:
memory needed, expected speed (with a link to the measurement), where the recipe comes from, and its status.

## Status labels

| Label | Meaning |
|---|---|
| **official** | Copied from an [NVIDIA DGX Spark playbook](https://github.com/NVIDIA/dgx-spark-playbooks). NVIDIA lists the model as tested for that engine. |
| **community-measured** | Copied from a community source that published a measured result for this exact setup. |
| **untested, from &lt;source&gt;** | We adapted it: swapped the model, changed flags, or turned a `docker run` into a compose file. The source did not publish a result for this exact command. |

Nobody has re-run these commands for this repo yet. If you have, open a [benchmark issue](../.github/ISSUE_TEMPLATE/benchmark.yml) and we will update the label.

All facts were checked on **2026-10-20**. Container tags and vLLM nightlies change fast, so compare with the linked source before you copy a command.

## Index

| Engine | Recipes |
|---|---|
| Ollama | [gpt-oss](ollama/gpt-oss.md) · [Qwen3 family](ollama/qwen3.md) · [Llama 3.3 70B](ollama/llama-3.3-70b.md) |
| vLLM | [gpt-oss-120b](vllm/gpt-oss-120b.md) · [gpt-oss-20b](vllm/gpt-oss-20b.md) · [Qwen3-30B-A3B / Qwen3-Coder-30B-A3B](vllm/qwen3-30b-a3b.md) · [Qwen3-Coder-Next](vllm/qwen3-coder-next.md) · [Llama 3.3 70B](vllm/llama-3.3-70b.md) |
| llama.cpp | [build](llama.cpp/build.md) · [gpt-oss-120b / 20b](llama.cpp/gpt-oss.md) · [Qwen3-Coder-30B-A3B](llama.cpp/qwen3-coder-30b-a3b.md) |
| SGLang | [gpt-oss-120b / 20b](sglang/gpt-oss.md) |
| TensorRT-LLM | [gpt-oss-120b (+ EAGLE-3)](tensorrt-llm/gpt-oss-120b.md) · [Llama 3.3 70B (+ draft model)](tensorrt-llm/llama-3.3-70b.md) |
| Two Sparks | [network, NCCL, vLLM, TRT-LLM, llama.cpp RPC](cluster/two-sparks.md) |
| Fine-tuning | [Unsloth QLoRA](finetune/unsloth-qlora.md) |
| UI | [Open WebUI](open-webui/open-webui.md) |
