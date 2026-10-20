# Benchmarks

[`benchmarks.csv`](benchmarks.csv) has 138 published measurements of LLM inference on GB10 hardware: DGX Spark, ASUS Ascent GX10, and 2× / 4× Spark clusters. Dates run from 2025-10-13 to 2026-08-10. Every row links to its source. **We did not run any of these ourselves.** We collected and normalised them on 2026-10-20.

## Columns

| Column | Meaning |
|---|---|
| `date` | Publication or measurement date as given by the source (`2026-08` when only the month is known) |
| `device`, `nodes` | GB10 system and number of nodes |
| `model`, `params_b`, `active_b` | Model; total and active parameters in billions (active = MoE experts used per token) |
| `quant` | Weight format. "(assumed)" means the source did not state it |
| `engine`, `engine_version` | Inference engine and build or tag, when published |
| `workload` | Prompt/output shape, e.g. `pp2048/tg32 @ depth 8192` = 2,048-token prompt and 32 generated tokens with 8,192 tokens already in the KV cache |
| `concurrency` | Parallel requests |
| `metric` | `decode` = output tokens/s (value in `decode_tps`); `total_in_out` = input + output tokens/s (value in `total_tps`). The two are not comparable |
| `decode_tps` | Generation (output) speed, tokens/s |
| `total_tps` | Total token throughput, input + output, tokens/s. Only StorageReview's vLLM rows (Equal ISL/OSL and Decode Heavy profiles) use it |
| `scope` | `per_request` when concurrency = 1; `aggregate` = total over all parallel requests |
| `prefill_tps` | Prompt processing, tokens/s (blank if not published) |
| `ttft_ms` | Time to first token |
| `reliability` | See below |
| `source_title`, `source_url`, `notes` | Where the number comes from and how it was measured |

## Reliability levels

| Level | Rule | Rows |
|---|---|---|
| `reproducible` | The exact tool and parameters are published (`llama-bench`, `llama-batched-bench`, `vllm bench serve`, `trtllm-bench`, `llama-benchy`, or a public harness) | 62 |
| `vendor` | Published by NVIDIA, Ollama, or LMSYS (SGLang) about their own stack. Method summarised, full command not always given | 36 |
| `reported` | A review or forum post. Numbers may be rounded, and the method is only partly described | 40 |

## What we left out of the source data

- **Non-GB10 hardware.** The research file also had RTX 5090, RTX PRO 6000, Apple M-series, Strix Halo and Jetson Thor numbers. Spark-vs-X comparisons belong in their own sources.
- **A table its author later called outdated** (eugr, [forum 2026-01-08](https://forums.developer.nvidia.com/t/dgx-spark-performance/356716)).
- **An unconfirmed outlier:** Qwen3.6-35B-A3B on the "Atlas" engine, 218.85 tok/s ([forum, 2026-05-26](https://forums.developer.nvidia.com/t/nemotron-3-super-120b-a12b-nvfp4-on-single-dgx-spark-23-45-tok-s-spark-arena-com-benhmarks/370070)). That is 2.2–2.8× the vLLM single-user figures for the same model, 78.2 tok/s without MTP and 101 tok/s with MTP ([field-notes results](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/benchmarks/results.jsonl)). Its author reports it halves at 2 concurrent requests.
- **Entries without a single number:** the gpt-oss-120b vLLM 8× aggregate is published only as a 116–153 range; a two-node Qwen3-VL-235B "~25 tok/s" has no stated conditions.
- **A stale changelog figure:** GLM-4.7-Flash AWQ "~40 t/s" was later re-measured at 23.5 tok/s by [field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md).
- **Prefill from the LMSYS Ollama rows**, which LMSYS's sheet shows as unstable. Decode kept.

## Caveats when reading the numbers

1. **Do not compare `per_request` with `aggregate`.** 262 tok/s in total for 32 users does not mean one user gets 262 tok/s.
2. **Do not compare `total_in_out` with `decode`.** StorageReview's vLLM numbers count input and output tokens together (Equal ISL/OSL profile), so they are not generation speed.
3. **Workloads differ between sources.** `tg32` (32 tokens), a 400-token completion and a 2,048-token completion give different numbers on the same machine. Compare rows from the same source first.
4. **Context depth matters.** In llama.cpp, gpt-oss-120b drops from 58.7 tok/s at empty context to 42.8 tok/s at 32k.
5. **Software moves fast.** gpt-oss-120b on llama.cpp went from 52.9 tok/s (build b6767, October 2025) to 58.7 tok/s (b7941, February 2026). vLLM numbers depend heavily on whether the image has sm_121 kernels: NVFP4 in emulation runs at ~1.1 tok/s ([forum](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766)).
6. **Ollama serialises requests with the default `OLLAMA_NUM_PARALLEL=1`.** In the field-notes runs, its 8-request aggregate equalled its single-request speed.
7. **Warm-up.** A cold first request measured 16 tok/s on a model that settled at ~53 tok/s ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md)). Sources differ on whether they discard warm-up runs.
8. **Driver and OS versions differ between rows.** Launch-era numbers used driver 580.95.05 and kernel 6.11; the field-notes rows used 580.173.02. See `notes`.
9. **GX10 vs Founders Edition.** Same GB10 chip and memory bandwidth, but power and cooling limits can differ. GX10 rows are labelled.

## Adding a measurement

See [CONTRIBUTING.md](../CONTRIBUTING.md). The short version: open a benchmark issue (template: [`.github/ISSUE_TEMPLATE/benchmark.yml`](../.github/ISSUE_TEMPLATE/benchmark.yml)) with the model, engine and version, quantisation, context, the exact command, and the raw tool output.
