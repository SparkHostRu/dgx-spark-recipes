# Contributing

The most useful contribution is **a measurement from your own Spark**, with enough detail that someone else can reproduce it. Recipe fixes, newer container tags and "this command no longer works" reports are just as welcome.

## Ground rules

- **Every number and every command needs a source.** Your own run counts if you include the command and the raw output.
- **Say what you ran.** If you changed a recipe (model, flags, image tag), say so. We label recipes `official`, `community-measured` or `untested`; see [recipes/README.md](recipes/README.md).
- **Do not paste large blocks of someone else's documentation.** Link to it, summarise it in your own words, and copy only the commands. If you copy a command from a source with a license not yet listed in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md), add it there.

## Adding a benchmark

Open an issue with the **Benchmark result** template, or send a PR that adds a row to [`benchmarks/benchmarks.csv`](benchmarks/benchmarks.csv). We need:

| Field | Example |
|---|---|
| Hardware | DGX Spark Founders Edition 128 GB / ASUS GX10 / 2× Spark over QSFP |
| DGX OS, driver, CUDA | DGX OS 7.5.0, driver 580.159.03, CUDA 13.0.2 (`nvidia-smi` shows driver and CUDA) |
| Model | Exact Hugging Face repo or Ollama tag, e.g. `openai/gpt-oss-120b`, `qwen3:30b-a3b` |
| Quantisation | MXFP4, NVFP4, Q4_K_M, FP8, AWQ… |
| Engine and version | `llama.cpp b7941`, `vllm 0.28.0`, image tag `nvcr.io/nvidia/vllm:26.09-py3` |
| Context / workload | `pp2048 tg32 @ depth 0`, or ISL/OSL, or prompt and max_tokens |
| Concurrency | 1, 8, 32… and whether the number is per request or the aggregate |
| Metric | Decode (output tokens/s) or total throughput (input + output tokens/s). Say which |
| Command | The full server command and the full benchmark command |
| Raw output | `llama-bench` table, `vllm bench serve` summary, or your harness's JSON |

### Recommended tools

```bash
# llama.cpp, current master (mmap off via -lm none)
llama-bench -m model.gguf -fa 1 -d 0,4096,8192,16384,32768 -p 2048 -n 32 -ub 2048 -lm none
llama-batched-bench -m model.gguf -fa 1 -c 300000 -ub 2048 -npp 4096,8192 -ntg 32 -npl 1,2,4,8,16,32 -lm none
# on older builds (e.g. b7941, used for the upstream DGX Spark numbers) the flag was -mmp 0 / --no-mmap

# any OpenAI-compatible server (vLLM, SGLang, TRT-LLM, llama-server)
vllm bench serve --help        # built into vLLM
# llama-benchy: llama-bench-style reports for any OpenAI-compatible backend
# https://github.com/eugr/llama-benchy
```

### Before you submit

- Discard a warm-up request.
- Check the server log for the intended kernel path. A fallback path (for example NVFP4 emulation) makes the number meaningless.
- Only one large model should be resident.
- vLLM: do not use `--enforce-eager` unless it is the point of the test.

## Adding or fixing a recipe

1. Copy the header table from an existing file in `recipes/<engine>/`.
2. Link the source of every command. Mark adapted commands *untested, from &lt;source&gt;*.
3. Give an expected speed only with a link to a measurement. Otherwise write "no published measurement found".
4. For container images, confirm that a `linux/arm64` variant exists.

## License

By contributing, you agree to license your contribution under the [MIT License](LICENSE).
