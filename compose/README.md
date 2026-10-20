# Compose files

| Stack | Folder | Images (arm64 manifests checked 2026-10-20) | Status |
|---|---|---|---|
| Open WebUI with bundled Ollama | [`ollama-openwebui/`](ollama-openwebui/compose.yaml) | `ghcr.io/open-webui/open-webui:ollama` | **untested**. Compose translation of the official [`open-webui` playbook](https://build.nvidia.com/spark/open-webui) `docker run` |
| vLLM (OpenAI API) + Open WebUI | [`vllm-openwebui/`](vllm-openwebui/compose.yaml) | `nvcr.io/nvidia/vllm:26.05.post1-py3`, `ghcr.io/open-webui/open-webui:main` | **untested**. Combines the serve command from the legacy [`vllm` playbook](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md) with Open WebUI's `OPENAI_API_BASE_URL` wiring from the [`trt-llm` playbook](https://build.nvidia.com/spark/trt-llm) |

We checked architectures by reading each image's manifest list in its registry (Docker Hub, ghcr.io, nvcr.io). Every image above has a `linux/arm64` entry. GB10 is aarch64, so an image without one will not run.

## Ollama + Open WebUI

```bash
cd compose/ollama-openwebui
docker compose up -d
# open http://<spark-ip>:8080, create the admin account, then pull a model from the UI,
# for example gpt-oss:20b or qwen3:30b-a3b
```

## vLLM + Open WebUI

```bash
cd compose/vllm-openwebui
cp .env.example .env      # set HF_TOKEN and MODEL
docker compose up -d
docker compose logs -f vllm      # wait for the server to report ready
curl -s http://localhost:8000/v1/models
# open http://<spark-ip>:8080
```

Notes:

- Requires the NVIDIA Container Toolkit, which DGX OS ships with. `deploy.resources.reservations.devices` is the compose equivalent of `--gpus all`.
- With the stock NGC image, gpt-oss runs on the default MXFP4 path. The fastest measured gpt-oss-120b setup (60.7 tok/s) needs a community image build; see [recipes/vllm/gpt-oss-120b.md](../recipes/vllm/gpt-oss-120b.md).
- The vLLM container downloads weights as root. Use `sudo` to clear the cache later (troubleshooting in the legacy vLLM playbook).
- Only one large model fits at a time. Stop other servers before you start this one.

If you run either file, please report the result in an issue. We will drop the "untested" label once someone has confirmed it on hardware.
