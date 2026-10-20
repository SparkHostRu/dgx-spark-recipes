# DGX Spark Recipes

**Copy-paste recipes for running LLMs on NVIDIA DGX Spark (GB10): which engine to use for each model, the exact commands, and the speed you can expect. Every number links to its source.**

[Русская версия](README.ru.md)

- Models: gpt-oss-120b / 20b, Qwen3 (30B-A3B, Coder 30B-A3B, Coder-Next), Llama 3.3 70B, Qwen3-235B on two Sparks
- Engines: Ollama, vLLM, llama.cpp, SGLang, TensorRT-LLM
- Also covered: two-Spark clusters, QLoRA fine-tuning with Unsloth, Open WebUI, docker compose files
- [`benchmarks/benchmarks.csv`](benchmarks/benchmarks.csv): 138 published GB10 measurements, each with a source link and a reliability level

Facts checked on **2026-10-20**. Each recipe has a status: **official** (NVIDIA playbook), **community-measured** (copied from a source that published a result), or **untested, from &lt;source&gt;** (adapted by us). See [recipes/README.md](recipes/README.md).

---

## Quick start: Ollama, about 2 minutes plus a 14 GB download

The install command comes from NVIDIA's [`ollama` playbook](https://build.nvidia.com/spark/ollama). `gpt-oss:20b` is the model Ollama itself benchmarked on Spark (58.3 tok/s, [Ollama blog](https://ollama.com/blog/nvidia-spark-performance)).

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama run gpt-oss:20b
```

For a web chat UI on port 8080 (NVIDIA [`open-webui` playbook](https://build.nvidia.com/spark/open-webui)):

```bash
docker run -d -p 8080:8080 --gpus=all \
  -v open-webui:/app/backend/data \
  -v open-webui-ollama:/root/.ollama \
  --name open-webui ghcr.io/open-webui/open-webui:ollama
```

---

## Which engine for which model

Decode speed for **one user** on **one Spark**, unless stated otherwise. "Many users" is the total throughput across all parallel requests.

| Model | Best measured engine | 1 user, tok/s | Many users, tok/s | Source (date) | Recipes |
|---|---|---|---|---|---|
| **gpt-oss-120b** | vLLM, MXFP4 CUTLASS build | **60.7** | 116–153 (8) | [ss121 (2026-08-10)](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | [vLLM](recipes/vllm/gpt-oss-120b.md) · [llama.cpp](recipes/llama.cpp/gpt-oss.md) · [Ollama](recipes/ollama/gpt-oss.md) · [SGLang](recipes/sglang/gpt-oss.md) · [TRT-LLM](recipes/tensorrt-llm/gpt-oss-120b.md) |
| | llama.cpp (no build needed) | 58.7 | 262.2 (32) | [llama.cpp benches (2026-02-05)](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | |
| **gpt-oss-20b** | llama.cpp | **83.4** | 473.6 (32) | [llama.cpp benches (2026-02-05)](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | [llama.cpp](recipes/llama.cpp/gpt-oss.md) · [vLLM](recipes/vllm/gpt-oss-20b.md) · [Ollama](recipes/ollama/gpt-oss.md) · [SGLang](recipes/sglang/gpt-oss.md) |
| **Qwen3-30B-A3B** | Ollama, Q4_K_M | **85.2** | 87.5 (8) | [ss121 (2026-08-10)](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | [Ollama](recipes/ollama/qwen3.md) · [vLLM](recipes/vllm/qwen3-30b-a3b.md) |
| | vLLM, NVFP4 | 77.1 | **313.4** (8) | same | |
| **Qwen3-Coder-30B-A3B** | llama.cpp, Q8_0 | **61.1** | 216.8 (32) | [llama.cpp benches (2026-02-05)](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | [llama.cpp](recipes/llama.cpp/qwen3-coder-30b-a3b.md) · [vLLM](recipes/vllm/qwen3-30b-a3b.md) · [Ollama](recipes/ollama/qwen3.md) |
| **Qwen3-Coder-Next** (80B-A3B) | vLLM, INT4 AutoRound | **74.8** | 289.0 (8) | [ss121 (2026-08-10)](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | [vLLM](recipes/vllm/qwen3-coder-next.md) · [Ollama](recipes/ollama/qwen3.md) |
| **Llama 3.3 70B** (dense) | Ollama, Q4_K_M | **~5** | — | [field-notes (2026-08)](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/hardware-model.md) | [Ollama](recipes/ollama/llama-3.3-70b.md) · [vLLM](recipes/vllm/llama-3.3-70b.md) · [TRT-LLM + draft](recipes/tensorrt-llm/llama-3.3-70b.md) |
| **gpt-oss-120b, 2× Spark** | vLLM 0.17.0 + MXFP4 patches, TP=2 | **81.5** | — | [forum (2026-03-11)](https://forums.developer.nvidia.com/t/vllm-0-17-0-mxfp4-patches-for-dgx-spark-qwen3-5-35b-a3b-70-tok-s-gpt-oss-120b-80-tok-s-tp-2/362824) | [two Sparks](recipes/cluster/two-sparks.md) |
| **Qwen3-235B-A22B, 2× Spark** | TensorRT-LLM, NVFP4, TP=2 | **21.65** | — | [forum (2025-12-19)](https://forums.developer.nvidia.com/t/question-on-inference-performance-results-of-qwen3-235b-a22b-on-2x-dgx-spark/355053) | [two Sparks](recipes/cluster/two-sparks.md) |

Rules of thumb from these numbers:

- **MoE beats dense.** Decode is limited by memory bandwidth (273 GB/s), and an MoE model only reads its active experts. gpt-oss-120b (5.1B active) runs about 8× faster than Llama 3.3 70B.
- **One user:** llama.cpp and Ollama start in seconds and are competitive. A well-built vLLM image was faster on 3 of 4 models in a same-harness comparison ([ss121](https://github.com/sergioamsilva/dgx-spark-field-notes)).
- **Several users, an API, or agents:** use vLLM or SGLang. Ollama does not batch: with 8 parallel requests its total throughput equals its single-request speed.

---

## Benchmarks

A selection from [`benchmarks/benchmarks.csv`](benchmarks/benchmarks.csv). The [methodology and caveats](benchmarks/README.md) explain how to read it. Most importantly, never compare per-request and aggregate numbers.

| Model | Engine | Quant | Decode tok/s (users) | Prefill tok/s | HW | Source | Date |
|---|---|---|---|---|---|---|---|
| gpt-oss-120b | llama.cpp b7941 | MXFP4 | 58.7 (1) · 42.8 at 32k ctx | 2,444 (pp2048) | 1× | [ggerganov](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | 2026-02-05 |
| gpt-oss-120b | vLLM, MXFP4 CUTLASS | MXFP4 | 60.7 (1) | — | 1× | [ss121](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | 2026-08-10 |
| gpt-oss-120b | SGLang (`sglang:spark`) | MXFP4 | 50.5 (1) · 203.1 (32) | — | 1× | [LMSYS](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark) | 2025-10-13 |
| gpt-oss-120b | Ollama 0.12.6 | MXFP4 | 41.1 (1) | 1,169 | 1× | [Ollama](https://ollama.com/blog/nvidia-spark-performance) | 2025-10-23 |
| gpt-oss-20b | llama.cpp b7941 | MXFP4 | 83.4 (1) · 473.6 (32) | 4,506 | 1× | [ggerganov](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | 2026-02-05 |
| Qwen3-Coder-30B-A3B | llama.cpp b7941 | Q8_0 | 61.1 (1) · 216.8 (32) | 2,987 | 1× | [ggerganov](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | 2026-02-05 |
| Qwen3-30B-A3B | vLLM | NVFP4 | 77.1 (1) · 313.4 (8) | — | 1× | [ss121](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | 2026-08-10 |
| Qwen3-Coder-Next | vLLM | INT4 AutoRound | 74.8 (1) · 289.0 (8) | — | 1× | [ss121](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | 2026-08-10 |
| Qwen3-14B | TensorRT-LLM | NVFP4 | 22.7 (1) | 5,929 | 1× | [NVIDIA](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks) | 2025-10-24 |
| Llama 3.1 70B | Ollama 0.12.6 | Q4_K_M | 4.4 (1) | 1,911 | 1× | [Ollama](https://ollama.com/blog/nvidia-spark-performance) | 2025-10-23 |
| Nemotron 3 Super 120B-A12B | vLLM nightly + MTP | NVFP4 | 23.45 (1) | — | 1× | [forum](https://forums.developer.nvidia.com/t/nemotron-3-super-120b-a12b-nvfp4-on-single-dgx-spark-23-45-tok-s-spark-arena-com-benhmarks/370070) | 2026-05-13 |
| gpt-oss-120b | vLLM 0.17.0 + patches, TP=2 | MXFP4 | 81.5 (1) | 6,069 | 2× | [forum](https://forums.developer.nvidia.com/t/vllm-0-17-0-mxfp4-patches-for-dgx-spark-qwen3-5-35b-a3b-70-tok-s-gpt-oss-120b-80-tok-s-tp-2/362824) | 2026-03-11 |
| Qwen3-235B-A22B | TensorRT-LLM 1.2.0rc5, TP=2 | NVFP4 | 21.65 (1) | — | 2× | [forum](https://forums.developer.nvidia.com/t/question-on-inference-performance-results-of-qwen3-235b-a22b-on-2x-dgx-spark/355053) | 2025-12-19 |
| Qwen3.5-397B-A17B | vLLM, TP=4 | INT4 AutoRound | ~37 (1) · ~103 (4) | — | 4× | [spark-vllm-docker](https://github.com/eugr/spark-vllm-docker) | 2026-03-12 |

Community leaderboard with a recipe for every result: [Spark Arena](https://spark-arena.com/leaderboard). We have not imported it yet.

---

## Official NVIDIA resources

- **Playbooks:** [build.nvidia.com/spark](https://build.nvidia.com/spark), with sources in [NVIDIA/dgx-spark-playbooks](https://github.com/NVIDIA/dgx-spark-playbooks). The ones this repo uses most: [ollama](https://build.nvidia.com/spark/ollama), [open-webui](https://build.nvidia.com/spark/open-webui), [vllm](https://build.nvidia.com/spark/vllm), [llama-cpp](https://build.nvidia.com/spark/llama-cpp), [sglang](https://build.nvidia.com/spark/sglang), [trt-llm](https://build.nvidia.com/spark/trt-llm), [speculative-decoding](https://build.nvidia.com/spark/speculative-decoding), [connect-two-sparks](https://build.nvidia.com/spark/connect-two-sparks), [nccl](https://build.nvidia.com/spark/nccl), [unsloth](https://build.nvidia.com/spark/unsloth), [nemo-fine-tune](https://build.nvidia.com/spark/nemo-fine-tune), [nvfp4-quantization](https://build.nvidia.com/spark/nvfp4-quantization)
- **Hardware spec:** [DGX Spark Hardware Overview](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)
- **DGX OS and firmware versions:** [DGX Spark Release Notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)
- **Forum:** [NVIDIA Developer Forums, DGX Spark / GB10](https://forums.developer.nvidia.com/)

Community resources: [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker) (vLLM images and per-model recipes, also referenced by NVIDIA's multi-node playbook) · [dgx-spark-field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes) (same-harness measurements, MIT) · [llama.cpp DGX Spark benches](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) · [llama-benchy](https://github.com/eugr/llama-benchy) · [Spark Arena](https://spark-arena.com/leaderboard)

---

## Gotchas

**Hardware in one line:** GB10 has 20 Arm cores and a Blackwell GPU (compute capability 12.1), sharing 128 GB of LPDDR5x at 273 GB/s ([spec](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)). From 2026-10-23, OEMs also ship a 64 GB version ([StorageReview](https://www.storagereview.com/news/nvidia-dgx-spark-64gb-october-23-4999-two-units-cluster-to-128gb)). This repo assumes 128 GB.

### 1. It is aarch64: use arm64 images

GB10 is ARM64. An image without a `linux/arm64` variant will not start. NGC images (`nvcr.io/nvidia/vllm`, `nvcr.io/nvidia/tensorrt-llm/release`, `nvcr.io/nvidia/pytorch`) are multi-arch, and so are `vllm/vllm-openai`, `lmsysorg/sglang:latest-cu130`, `ollama/ollama` and `ghcr.io/open-webui/open-webui`. We checked their registry manifests on 2026-10-20. To check another image: `docker manifest inspect <image> | grep arm64`.

### 2. Unified memory behaves differently

- CPU and GPU share the same 128 GB. About **117–121 GiB** is usable for models ([forum](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126), [field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes)). Display memory takes 2 GB by default and can be set to 4 GB in BIOS since July 2026 ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)).
- `nvidia-smi --query-gpu` shows `N/A` for memory fields. That is expected on UMA; use plain `nvidia-smi` (NVIDIA vLLM playbook).
- The page cache is not handed back to the GPU automatically. If a model will not load although memory looks free, run `sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'` (most NVIDIA playbooks). In `free`, read the `available` column, not `used`.
- Keep one large model resident at a time.
- NIM containers cap `gpu_memory_utilization` at 0.50 on UMA, so ~75 GB models fail with OOM. Use plain `vllm/vllm-openai` instead ([forum](https://forums.developer.nvidia.com/t/nemotron-3-super-120b-a12b-nvfp4-on-single-dgx-spark-23-45-tok-s-spark-arena-com-benhmarks/370070)).

### 3. sm_121 and CUDA 13

- Build flags: llama.cpp `-DCMAKE_CUDA_ARCHITECTURES=121a-real`, NCCL `NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"` (NVIDIA playbooks).
- Use CUDA 13 builds: `lmsysorg/sglang:latest-cu130`, PyTorch wheels from `.../whl/cu130`.
- **NVFP4 without sm_121 kernels falls back to emulation and runs at ~1.1 tok/s.** With FlashInfer CUTLASS kernels the same weights run at 77.1 tok/s. Look for `Using FlashInferCutlassNvFp4LinearKernel` or `[MXFP4] Using backend: CUTLASS` in the log ([forum](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766), [field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md)).
- vLLM: do not use `--enforce-eager`. It disables CUDA graphs, and NVFP4 dropped from 67 to 23 tok/s with it (field-notes).

### 4. OOM, swap and hangs

- If a model over-commits unified memory, the whole machine can stall instead of the process failing cleanly. For this reason the community cluster launcher has `--earlyoom`, which kills vLLM first ([vllm/multi-node playbook](https://build.nvidia.com/spark/vllm/multi-node)). DGX OS improved OOM handling on UMA in July 2026 ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)).
- Most recipes here use `--gpu-memory-utilization` 0.7–0.8. Leave headroom for the OS.
- **Swap:** we found no NVIDIA guidance on swap sizing for Spark. Do not count on swap to fit a model that does not fit in memory. Size the model and KV cache to the `available` memory.

### 5. DGX OS, driver and firmware versions

- Current Founders Edition stack ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html), read 2026-10-20): **DGX OS 7.5.0** (Ubuntu 24.04), **driver 580.159.03**, **CUDA 13.0.2**, kernel 6.17.
- Since June 2026, OTA updates are not installed by default during first setup. Update a freshly set-up machine before you benchmark it.
- A February 2026 update fixed a multi-Spark performance regression introduced after DGX OS 7.4.0.
- OEM GB10 systems (ASUS, Dell, HP, Lenovo, …) may get firmware and OS updates later than the Founders Edition.
- spark-vllm-docker reports that **driver 590.x deadlocks during CUDAGraph capture on GB10 unified memory** and requires 580.x for its Qwen3.5-397B recipe ([README](https://github.com/eugr/spark-vllm-docker), 2026-03-12).

### 6. Smaller things that waste an afternoon

- **Slow model loading with mmap:** use `--no-mmap` in llama.cpp (costs ~20 GB extra) and/or raise NVMe read-ahead to 8192 KB ([llama.cpp #16578](https://github.com/ggml-org/llama.cpp/discussions/16578), [field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/loading-performance.md)).
- **Empty `content` with `finish_reason: length`:** a thinking model used up `max_tokens` on reasoning. Raise `max_tokens` (NVIDIA playbooks).
- **`PermissionError` on `~/.cache/huggingface` or a cache you cannot delete:** containers download as root. Use `sudo chown -R "$USER:$USER" ~/.cache/huggingface`.
- **A Spark shuts down under heavy load:** `sudo nvidia-smi -lgc 200,2150` lowers the GPU clock; it resets on reboot ([spark-vllm-docker](https://github.com/eugr/spark-vllm-docker)).
- **The first request is slow:** a cold request measured 16 tok/s on a model that settled at ~53. Throw away a warm-up request before you measure.

---

## Repository layout

```
recipes/            one file per engine × model; status label in each header
  ollama/ vllm/ llama.cpp/ sglang/ tensorrt-llm/
  cluster/two-sparks.md
  finetune/unsloth-qlora.md
  open-webui/open-webui.md
compose/            vLLM + Open WebUI, Ollama + Open WebUI
benchmarks/         benchmarks.csv + methodology
```

## Contributing

Measured something on your Spark? Open a **Benchmark result** issue. The template asks for the model, engine and version, quantisation, context, the exact command, and the raw `llama-bench` / `vllm bench serve` output. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE). Commands quoted from NVIDIA playbooks and community repositories are reproduced for reference and remain under their original licenses; every one links to its source.

---

<sub>Need a DGX Spark without buying one? Rent it at [SparkHost](https://sparkhost.ru) (Moscow).</sub>
