# Build llama.cpp for GB10 (sm_121)

| | |
|---|---|
| Recipe source | NVIDIA playbook [`llama-cpp`](https://build.nvidia.com/spark/llama-cpp), Last Updated 10/09/2026 |
| Status | **official** |
| Build time | 5–10 minutes (playbook) |

```bash
sudo apt update
sudo apt install -y git clang cmake libcurl4-openssl-dev libssl-dev

git clone https://github.com/ggml-org/llama.cpp ~/llama.cpp
cd ~/llama.cpp

cmake -B build -DGGML_NATIVE=ON -DGGML_CUDA=ON -DGGML_CURL=ON -DGGML_RPC=ON -DCMAKE_CUDA_ARCHITECTURES=121a-real
cmake --build build --config Release --target llama-server -j
```

- `121a-real` targets compute capability 12.1, which is GB10.
- `-DGGML_RPC=ON` lets you split one model across two Sparks: [cluster/two-sparks.md](../cluster/two-sparks.md#3d-llamacpp-rpc-community).
- To build the benchmark tools as well, drop `--target llama-server`.

## Health check (from the playbook)

```bash
timeout 900 bash -c 'until curl -sf http://127.0.0.1:30000/health > /dev/null 2>&1; do sleep 5; done' || exit 1
```

## Benchmark the way upstream does

The [llama.cpp DGX Spark numbers](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) ([discussion #16578](https://github.com/ggml-org/llama.cpp/discussions/16578)) were produced on build b7941 with `-mmp 0` / `--no-mmap`. Those two flags no longer exist on current llama.cpp `master`: mmap is now controlled by `-lm` / `--load-mode` (`-lm none` = no mmap). Pick the variant that matches your goal.

**Reproduce the published numbers** (build b7941, original flags):

```bash
cd ~/llama.cpp && git checkout b7941
cmake -B build -DGGML_NATIVE=ON -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=121a-real
cmake --build build --config Release -j

./build/bin/llama-bench -m [model.gguf] -fa 1 -d 0,4096,8192,16384,32768 -p 2048 -n 32 -ub 2048 -mmp 0
./build/bin/llama-batched-bench -m [model.gguf] -fa 1 -c 300000 -ub 2048 -npp 4096,8192 -ntg 32 -npl 1,2,4,8,16,32 --no-mmap
```

**Current master** (*modified from the upstream commands*: `-mmp 0` / `--no-mmap` replaced by `-lm none`; untested):

```bash
./build/bin/llama-bench -m [model.gguf] -fa 1 -d 0,4096,8192,16384,32768 -p 2048 -n 32 -ub 2048 -lm none
./build/bin/llama-batched-bench -m [model.gguf] -fa 1 -c 300000 -ub 2048 -npp 4096,8192 -ntg 32 -npl 1,2,4,8,16,32 -lm none
```

The cmake flags in the b7941 block are the playbook's, minus the `llama-server`-only target, so that the bench tools are built too.

## Gotchas

- **Loading is slow with mmap on DGX OS.** Disable mmap: `-lm none` on current master, `--no-mmap` (`-mmp 0` in llama-bench) on older builds such as b7941. In [#16578](https://github.com/ggml-org/llama.cpp/discussions/16578) (2025-11-02), eugr suspects the kernel option `NO_PAGE_MAPCOUNT` in 6.17. Field-notes estimate that disabling mmap costs ~20 GB of extra memory, and also suggest raising NVMe read-ahead to 8192 KB ([loading-performance](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/loading-performance.md)).
- By default `llama-server` tries to allocate the full context and 4 parallel slots. Lower `--ctx-size` if you hit OOM (playbook).
