# Two Sparks: network, NCCL, and serving one model across both

Two Sparks connect directly with one QSFP cable through the ConnectX-7 NICs (up to 200 Gbit/s); no switch is needed ([NVIDIA playbook `connect-two-sparks`](https://build.nvidia.com/spark/connect-two-sparks)). Since June 2026, NVIDIA also supports a 3-node ring and up to 4 nodes through a switch ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)).

| What you get | Measured result |
|---|---|
| gpt-oss-120b, patched vLLM 0.17.0, TP=2 | **81.5 tok/s** decode, 6,069 tok/s prefill (pp2048) ([forum, 2026-03-11](https://forums.developer.nvidia.com/t/vllm-0-17-0-mxfp4-patches-for-dgx-spark-qwen3-5-35b-a3b-70-tok-s-gpt-oss-120b-80-tok-s-tp-2/362824)) |
| Qwen3-235B-A22B NVFP4, TensorRT-LLM, TP=2 | **21.65 tok/s** (trtllm-bench, ISL/OSL 128) ([forum, 2025-12-19](https://forums.developer.nvidia.com/t/question-on-inference-performance-results-of-qwen3-235b-a22b-on-2x-dgx-spark/355053)) |
| Qwen3-235B-A22B UD-Q4_K_XL, llama.cpp RPC | **~12.5 tok/s** (one Spark: ~1.8 tok/s) ([forum, 2025-12-18](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126)) |
| NCCL tests, community | Thread author: 39.34 GB/s (questioned in the thread as exceeding a 200 Gb/s link; a reply suggests it counts both directions). Another participant (#16): `gather_perf` ~22 GB/s ([same thread](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126)) |

## 1. Network: official

Source: playbook [`connect-two-sparks`](https://build.nvidia.com/spark/connect-two-sparks) ([GitHub](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/connect-two-sparks/README.md)), Last Updated 7/21/2026. Status: **official**, except the `curl` line that downloads `discover-sparks`: the playbook only links the script, so that line is *modified from* the playbook (we added the download, using the script's path in the repo).

```bash
ibdev2netdev     # shows which interfaces are Up

# Node 1
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 192.168.100.10/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.101.10/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
# Node 2: the same file with .11 instead of .10

# Passwordless SSH between the nodes (NVIDIA script)
curl -fsSL -o discover-sparks https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/connect-two-sparks/assets/discover-sparks
chmod +x discover-sparks
bash ./discover-sparks
```

- Both nodes need the same username.
- Each physical QSFP port appears as **two** logical interfaces (`enp1s0f1np1` and `enP2p1s0f1np1`).
- One cable gives full bandwidth. With two cables, all four interfaces need IP addresses (playbook).

## 2. NCCL check: official

Source: [`nvidia/playbook-nccl/README.md`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/playbook-nccl/README.md) in the NVIDIA playbooks repo, Last Updated 10/14/2026. The published page [build.nvidia.com/spark/nccl](https://build.nvidia.com/spark/nccl) still shows an older revision (12/15/2025, NCCL `v2.30.7-1`, assets under `nvidia/nccl/`). Status: **official**.

```bash
curl -fsSL "https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/playbook-nccl/assets/setup.sh" -o setup.sh
curl -fsSL "https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/playbook-nccl/assets/launch.sh" -o launch.sh
bash setup.sh <NODE_2_IP>
bash launch.sh --topology direct <NODE_1_IP> <NODE_2_IP>
```

To build NCCL by hand for sm_121: `make -j src.build NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"` (NCCL `v2.31.2-1`; nccl-tests `v2.19.7`, because v2.20+ does not link against the system OpenMPI, according to the playbook).

## 3a. vLLM, Ray + NGC: official

Source: legacy NVIDIA playbook [`nvidia/vllm/README.md`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md), "Run on two Sparks", Last Updated 06/12/2026. The current page [build.nvidia.com/spark/vllm](https://build.nvidia.com/spark/vllm) was reorganised on 09/14/2026 around NVIDIA Sync and points to [recipes.vllm.ai](https://recipes.vllm.ai/browse?panel=open&hw=dgx_spark_gb10) and spark-vllm-docker (section 3b); these Ray/NGC commands are now only in the GitHub README. Status: **official**.

```bash
# Both nodes. The playbook pins run_cluster.sh to a known-good commit
wget https://raw.githubusercontent.com/vllm-project/vllm/51c1ee9b7c8acbba4899a8ebffd390685d171946/examples/ray_serving/run_cluster.sh
# The 26.05-py3 NGC image ships without ray
sed -i 's|^RAY_START_CMD="ray start|RAY_START_CMD="pip install -q --root-user-action=ignore '\''ray[default]>=2.9'\'' \&\& ray start|' run_cluster.sh
chmod +x run_cluster.sh

docker pull nvcr.io/nvidia/vllm:26.05-py3
export VLLM_IMAGE=nvcr.io/nvidia/vllm:26.05-py3

# Node 1 (head), inside tmux/screen
export MN_IF_NAME=enp1s0f1np1
export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
bash run_cluster.sh $VLLM_IMAGE $VLLM_HOST_IP --head ~/.cache/huggingface \
  -e VLLM_HOST_IP=$VLLM_HOST_IP \
  -e UCX_NET_DEVICES=$MN_IF_NAME \
  -e NCCL_SOCKET_IFNAME=$MN_IF_NAME \
  -e OMPI_MCA_btl_tcp_if_include=$MN_IF_NAME \
  -e GLOO_SOCKET_IFNAME=$MN_IF_NAME \
  -e TP_SOCKET_IFNAME=$MN_IF_NAME \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=$VLLM_HOST_IP

# Node 2 (worker)
export MN_IF_NAME=enp1s0f1np1
export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
export HEAD_NODE_IP=<NODE_1_IP_ADDRESS>     # Node 1's QSFP-side IP (echo $VLLM_HOST_IP on Node 1)
export VLLM_IMAGE=nvcr.io/nvidia/vllm:26.05-py3
bash run_cluster.sh $VLLM_IMAGE $HEAD_NODE_IP --worker ~/.cache/huggingface \
  -e VLLM_HOST_IP=$VLLM_HOST_IP \
  -e UCX_NET_DEVICES=$MN_IF_NAME \
  -e NCCL_SOCKET_IFNAME=$MN_IF_NAME \
  -e OMPI_MCA_btl_tcp_if_include=$MN_IF_NAME \
  -e GLOO_SOCKET_IFNAME=$MN_IF_NAME \
  -e TP_SOCKET_IFNAME=$MN_IF_NAME \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=$HEAD_NODE_IP

# Head node: check the cluster
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
docker exec $VLLM_CONTAINER ray status

# Head node: download the model. Llama 3.3 is gated: accept the license on Hugging Face first.
# run_cluster.sh does not pass HF_TOKEN into the container, so log in inside it.
docker exec -it $VLLM_CONTAINER /bin/bash -c '
  hf auth login
  hf download meta-llama/Llama-3.3-70B-Instruct'

# Head node: serve
docker exec -it $VLLM_CONTAINER /bin/bash -c '
  vllm serve meta-llama/Llama-3.3-70B-Instruct \
    --tensor-parallel-size 2 --max-model-len 2048 \
    --gpu-memory-utilization 0.8 --distributed-executor-backend ray'
```

The playbook also runs Llama 3.1 405B AWQ-INT4 on two Sparks, but flags it as having "insufficient memory headroom for production use".

## 3b. vLLM, spark-vllm-docker: official pointer to a community repo

Source: playbook [`vllm/multi-node`](https://build.nvidia.com/spark/vllm/multi-node). NVIDIA's own multi-node page uses the community repo [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker). Status: **official**.

```bash
git clone https://github.com/eugr/spark-vllm-docker.git
cd spark-vllm-docker
curl -LsSf https://astral.sh/uv/install.sh | sh
./hf-download.sh nvidia/Qwen3.8-27B-NVFP4 -c --copy-parallel
./run-recipe.sh recipes/qwen3.8-27b-nvfp4-dflash2.yaml --setup
```

`launch-cluster.sh --earlyoom` kills vLLM before memory pressure hangs the system (multi-node page). The same repo holds recipes for the Qwen3 models above: `qwen3-coder-next-fp8` (TP=2 by default) and `openai-gpt-oss-120b` (solo only). See [vllm/qwen3-coder-next.md](../vllm/qwen3-coder-next.md).

## 3c. TensorRT-LLM, Qwen3-235B-A22B NVFP4: official

Source: playbook [`trt-llm`](https://build.nvidia.com/spark/trt-llm), "Run on two Sparks", Last Updated 04/28/2026. Status: **official**. Measured at 21.65 tok/s by a forum user on `1.2.0rc5` (see the table above).

```bash
cat > ~/openmpi-hostfile <<EOF
192.168.1.10
192.168.1.11
EOF

  docker run -d --rm \
  --name trtllm-multinode \
  --gpus '"device=all"' \
  --network host \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  --device /dev/infiniband:/dev/infiniband \
  -e UCX_NET_DEVICES="enp1s0f0np0,enp1s0f1np1" \
  -e NCCL_SOCKET_IFNAME="enp1s0f0np0,enp1s0f1np1" \
  -e OMPI_MCA_btl_tcp_if_include="enp1s0f0np0,enp1s0f1np1" \
  -e OMPI_MCA_orte_default_hostfile="/etc/openmpi-hostfile" \
  -e OMPI_MCA_rmaps_ppr_n_pernode="1" \
  -e OMPI_ALLOW_RUN_AS_ROOT="1" \
  -e OMPI_ALLOW_RUN_AS_ROOT_CONFIRM="1" \
  -e CPATH=/usr/local/cuda/include \
  -e TRITON_PTXAS_PATH=/usr/local/cuda/bin/ptxas \
  -v ~/.cache/huggingface/:/root/.cache/huggingface/ \
  -v ~/.ssh:/tmp/.ssh:ro \
  nvcr.io/nvidia/tensorrt-llm/release:1.3.0rc5 \
  sh -c "curl https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/trt-llm/assets/trtllm-mn-entrypoint.sh | sh"

docker cp ~/openmpi-hostfile trtllm-multinode:/etc/openmpi-hostfile
export TRTLLM_MN_CONTAINER=trtllm-multinode

docker exec $TRTLLM_MN_CONTAINER bash -c 'cat <<EOF > /tmp/extra-llm-api-config.yml
print_iter_log: false
kv_cache_config:
  dtype: "auto"
  free_gpu_memory_fraction: 0.9
cuda_graph_config:
  enable_padding: true
EOF'

docker exec \
  -e MODEL="nvidia/Qwen3-235B-A22B-FP4" \
  -e HF_TOKEN=$HF_TOKEN \
  -it $TRTLLM_MN_CONTAINER bash -c 'mpirun -x HF_TOKEN bash -c "hf download $MODEL"'

docker exec \
  -e MODEL="nvidia/Qwen3-235B-A22B-FP4" \
  -e HF_TOKEN=$HF_TOKEN \
  -it $TRTLLM_MN_CONTAINER bash -c '
    mpirun -x HF_TOKEN trtllm-llmapi-launch trtllm-serve $MODEL \
      --tp_size 2 \
      --backend pytorch \
      --max_num_tokens 32768 \
      --max_batch_size 4 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml \
      --port 8355'
```

- The two-Spark path uses image `1.3.0rc5`; the single-Spark path uses `1.3.0rc13` (as in the playbook).
- `ptxas fatal`: add `-x TRITON_PTXAS_PATH` to `mpirun`. The entrypoint will not start if port 2233 is taken.

## 3d. llama.cpp RPC: community

Source: [forum, 2025-12-18](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126). Status: **community-measured** (~12.5 tok/s, Qwen3-235B UD-Q4_K_XL, QSFP 200GbE, MTU 9000).

```bash
cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON -DLLAMA_CURL=OFF   # then a Release build (LLAMA_CURL is deprecated on current master)
./rpc-server -H <NODE_IP> -p 50052                               # on each node
./llama-server -m <model part 1> --rpc "169.254.1.2:50052" -ngl 999 -fit off --port 8082 -c 2048
```

`-fit off` avoids a segfault during memory fitting (author's note).

## Cluster gotchas

| Symptom | Fix | Source |
|---|---|---|
| Docker on node 2 cannot see the GPU (`Failed to initialize NVML`) | Copy `/etc/docker/daemon.json` from node 1 and restart Docker | [forum](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126) |
| Ray: "Current node has no GPU available" | Start the container with `--gpus=all`, host network, privileged (or pass the IB device through) | same thread (eugr) |
| Slow despite a working cluster | Plain Ethernet without the IB/RoCE devices costs a lot of performance | same thread |
| One Spark shuts down under heavy load | Lower the GPU clock: `sudo nvidia-smi -lgc 200,2150` (resets on reboot) | [spark-vllm-docker README](https://github.com/eugr/spark-vllm-docker) |
| Multi-Spark performance regression after DGX OS 7.4.0 | Fixed in a February 2026 update; stay up to date | [release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html) |
