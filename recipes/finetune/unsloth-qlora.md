# QLoRA fine-tuning with Unsloth

| | |
|---|---|
| What | 4-bit QLoRA (LoRA adapters on a 4-bit base model) with [Unsloth](https://github.com/unslothai/unsloth) |
| Memory | gpt-oss-120b QLoRA 4-bit: **~68 GB** of unified memory ([Unsloth, DGX Spark guide](https://unsloth.ai/docs/blog/fine-tuning-llms-with-nvidia-dgx-spark-and-unsloth)) |
| Expected speed | No Unsloth tok/s on Spark published. NVIDIA's own PyTorch numbers (not Unsloth, seq len 2048): Llama 3.3 70B QLoRA batch 8 peaks at 759.79 tok/s in their table; Llama 3.1 8B LoRA 6,969.59 tok/s ([NVIDIA blog, 2025-10-24](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks)). The text of the same post gives higher figures (5,079 and 53,658 tok/s) without explaining the difference |
| Recipe source | A: NVIDIA playbook [`unsloth`](https://build.nvidia.com/spark/unsloth) (legacy, Last Updated 12/15/2025). B: [Unsloth DGX Spark guide](https://unsloth.ai/docs/blog/fine-tuning-llms-with-nvidia-dgx-spark-and-unsloth) |
| Status | A: **official** (runs NVIDIA's smoke test). B: **vendor** (Unsloth's own guide). The "your own model" section: **untested** |

NVIDIA's newer [`playbook-fine-tuning`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/playbook-fine-tuning/README.md) (08/03/2026) no longer contains commands. It points to the Unsloth docs and warns readers not to invent package pins or container tags from it. Pins below are exactly what each source used, so check the source for newer ones.

## Path A: NVIDIA playbook (quick smoke test)

```bash
docker pull nvcr.io/nvidia/pytorch:25.11-py3
docker run --gpus all --ulimit memlock=-1 -it --ulimit stack=67108864 --entrypoint /usr/bin/bash --rm nvcr.io/nvidia/pytorch:25.11-py3

# inside the container
pip install transformers peft hf_transfer "datasets==4.3.0" "trl==0.26.1"
pip install --no-deps unsloth unsloth_zoo bitsandbytes
curl -O https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/unsloth/assets/test_unsloth.py
python test_unsloth.py
```

`test_unsloth.py` does QLoRA: it loads `unsloth/Phi-3.5-mini-instruct` with `load_in_4bit = True`, attaches LoRA (r=16, all attention and MLP projections), and trains with TRL's `SFTTrainer` on the LAION OIG `unified_chip2` dataset.

## Path B: Unsloth's DGX Spark image (notebooks, RL, gpt-oss)

```bash
sudo apt update && sudo apt install -y wget
wget -O Dockerfile "https://raw.githubusercontent.com/unslothai/notebooks/main/Dockerfile_DGX_Spark"
docker build -f Dockerfile -t unsloth-dgx-spark .

docker run -it \
    --gpus=all \
    --net=host \
    --ipc=host \
    --ulimit memlock=-1 \
    --ulimit stack=67108864 \
    -v $(pwd):$(pwd) \
    -v $HOME/.cache/huggingface:/root/.cache/huggingface \
    -w $(pwd) \
    unsloth-dgx-spark

# inside the container: the gpt-oss-20b RL example from the guide
NOTEBOOK_URL="https://raw.githubusercontent.com/unslothai/notebooks/refs/heads/main/nb/gpt_oss_(20B)_Reinforcement_Learning_2048_Game_DGX_Spark.ipynb"
wget -O "gpt_oss_20B_RL_2048_Game.ipynb" "$NOTEBOOK_URL"
jupyter notebook --ip=0.0.0.0 --port=8888 --no-browser --allow-root
```

When we checked (2026-10-20), [`Dockerfile_DGX_Spark`](https://github.com/unslothai/notebooks/blob/main/Dockerfile_DGX_Spark) was based on `nvcr.io/nvidia/pytorch:25.11-py3` (the guide page still shows an older copy based on 25.09-py3). It builds Triton and xformers from source for Blackwell (`TORCH_CUDA_ARCH_LIST="12.1"`), so expect a long first build. The guide says all Unsloth notebooks work on Spark once you remove their install cells, including the gpt-oss-120b notebook.

## Your own model: untested

In `test_unsloth.py`, change `model_name` to a 4-bit Unsloth checkpoint and point the dataset at your data. The script's own list of supported 4-bit models includes `unsloth/Meta-Llama-3.1-8B-Instruct-bnb-4bit` and `unsloth/Llama-3.3-70B-Instruct-bnb-4bit`. Neither has a published Unsloth measurement on Spark. *(untested, from test_unsloth.py)*

## Notes

- If training or loading hits OOM even though memory looks free, the page cache is probably holding it. Run `sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'` (NVIDIA `unsloth` playbook troubleshooting).
- Alternatives from NVIDIA: [`nemo-fine-tune`](https://build.nvidia.com/spark/nemo-fine-tune) (NeMo AutoModel, LoRA and QLoRA, including 70B) and [`pytorch-fine-tune`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/README.md) (FSDP + LoRA up to 70B on two Sparks, as announced in [NVIDIA's CES 2026 post](https://developer.nvidia.com/blog/new-software-and-model-optimizations-supercharge-nvidia-dgx-spark)).
- To serve the result, export it with Unsloth's save functions and run it on any engine in this repo.
