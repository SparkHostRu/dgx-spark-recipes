# Open WebUI on DGX Spark

| | |
|---|---|
| Images | `ghcr.io/open-webui/open-webui:ollama` (bundled Ollama) · `ghcr.io/open-webui/open-webui:main` (for an external OpenAI-compatible server). Both have linux/arm64 manifests (checked 2026-10-20) |
| Recipe source | NVIDIA playbook [`open-webui`](https://build.nvidia.com/spark/open-webui), Last Updated 10/14/2026. TRT-LLM wiring: playbook [`trt-llm`](https://build.nvidia.com/spark/trt-llm) |
| Status | **official** |

## With bundled Ollama

```bash
docker pull ghcr.io/open-webui/open-webui:ollama
docker run -d -p 8080:8080 --gpus=all \
  -v open-webui:/app/backend/data \
  -v open-webui-ollama:/root/.ollama \
  --name open-webui ghcr.io/open-webui/open-webui:ollama
```

Open `http://<spark-ip>:8080`. When launched through NVIDIA Sync, the playbook maps port `12000:8080` instead.

## In front of vLLM, SGLang or TensorRT-LLM

Use the `:main` image and point `OPENAI_API_BASE_URL` at the engine's `/v1` endpoint. The TRT-LLM playbook does this with `--network host` and `OPENAI_API_BASE_URL="http://localhost:8355/v1"`. For vLLM, the default port is 8000; for SGLang in our recipe, 30000.

A ready compose file for vLLM + Open WebUI is in [compose/vllm-openwebui](../../compose/vllm-openwebui/) (untested as compose).
