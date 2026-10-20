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

These are the commands behind the [llama.cpp DGX Spark numbers](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) ([discussion #16578](https://github.com/ggml-org/llama.cpp/discussions/16578)):

```bash
llama-bench -m [model.gguf] -fa 1 -d 0,4096,8192,16384,32768 -p 2048 -n 32 -ub 2048 -mmp 0
llama-batched-bench -m [model.gguf] -fa 1 -c 300000 -ub 2048 -npp 4096,8192 -ntg 32 -npl 1,2,4,8,16,32 --no-mmap
```

## Gotchas

- **Loading is slow with mmap on DGX OS.** Use `--no-mmap` / `-mmp 0`. In [#16578](https://github.com/ggml-org/llama.cpp/discussions/16578), eugr suspects the kernel option `NO_PAGE_MAPCOUNT` in 6.17. Field-notes estimate that `--no-mmap` costs ~20 GB of extra memory, and also suggest raising NVMe read-ahead to 8192 KB ([loading-performance](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/loading-performance.md)).
- By default `llama-server` tries to allocate the full context and 4 parallel slots. Lower `--ctx-size` if you hit OOM (playbook).
