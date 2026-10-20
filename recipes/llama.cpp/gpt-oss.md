# gpt-oss-120b and gpt-oss-20b on llama.cpp

| | |
|---|---|
| Models | [`ggml-org/gpt-oss-120b-GGUF`](https://huggingface.co/ggml-org/gpt-oss-120b-GGUF) (59.02 GiB) · [`ggml-org/gpt-oss-20b-GGUF`](https://huggingface.co/ggml-org/gpt-oss-20b-GGUF) (11.27 GiB) |
| Expected speed, 1 user (tg32 / pp2048) | **120b: 58.7 tok/s decode, 2,444 tok/s prefill** at empty context; 42.8 / 1,567 at 32k depth · **20b: 83.4 / 4,506**; 61.7 / 2,689 at 32k ([ggerganov, build b7941, 2026-02-05](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md)) |
| Expected speed, many users | `llama-batched-bench` with a 4,096-token prompt: 120b 139.1 tok/s in total with 8 users, 262.2 with 32 · 20b 233.1 with 8, 473.6 with 32 (same source) |
| Recipe source | Server command: ggerganov in [llama.cpp #16578](https://github.com/ggml-org/llama.cpp/discussions/16578) (2025-10-14), used for an AIME25 check (93.75 % at reasoning effort high) |
| Status | 120b: **community-measured, abridged** (flags removed, see below). 20b: **untested, from #16578** (model name changed) |

Build first: [build.md](build.md).

## Run

```bash
# gpt-oss-120b (abridged from #16578)
./build/bin/llama-server -hf ggml-org/gpt-oss-120b-GGUF --ctx-size 1048576 -np 8 --jinja -ub 2048 -b 2048 -ngl 99

# gpt-oss-20b (same command, model changed; untested)
./build/bin/llama-server -hf ggml-org/gpt-oss-20b-GGUF --ctx-size 1048576 -np 8 --jinja -ub 2048 -b 2048 -ngl 99
```

**Abridged:** compared with the original command in #16578 we removed these flags:

- sampling: `--temp 1.0 --top-p 1.0 --top-k 0 --min-p 0.01`
- reasoning effort: `--chat-template-kwargs '{"reasoning_effort": "high"}'`
- `--port 8066`, which was the port of the eval run
- `--no-mmap`. This flag does not exist on current llama.cpp master; use `-lm none` there

To reproduce the AIME25 setup, add the sampling and reasoning flags back.

- `--ctx-size 1048576 -np 8` gives 8 parallel slots of 131,072 tokens each. For a single user, use `-np 1` and a smaller `--ctx-size`.
- Add `--host 0.0.0.0 --port 30000` to reach the server from other machines (the playbook uses those values).
- If loading is slow, disable mmap: `-lm none` on current master, `--no-mmap` on older builds ([build.md](build.md#gotchas)).

## Reproduce the numbers

```bash
# on build b7941, as published
./build/bin/llama-bench -m <path/to/gpt-oss-120b-MXFP4.gguf> -fa 1 -d 0,4096,8192,16384,32768 -p 2048 -n 32 -ub 2048 -mmp 0
# on current master (-mmp was replaced by -lm)
./build/bin/llama-bench -m <path/to/gpt-oss-120b-MXFP4.gguf> -fa 1 -d 0,4096,8192,16384,32768 -p 2048 -n 32 -ub 2048 -lm none
```

`-hf` downloads into the llama.cpp cache. Point `-m` at the model file: `ggml-org/gpt-oss-120b-GGUF` now ships a single `gpt-oss-120b-MXFP4.gguf`. Older revisions were split into shards; in that case use the first shard. To check out b7941, see [build.md](build.md#benchmark-the-way-upstream-does).
