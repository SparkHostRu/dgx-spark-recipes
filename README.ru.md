# DGX Spark Recipes

**Готовые рецепты запуска LLM на NVIDIA DGX Spark (GB10): какой движок брать под какую модель, точные команды и ожидаемая скорость. У каждой цифры есть ссылка на источник.**

[English version](README.md)

- Модели: gpt-oss-120b / 20b, Qwen3 (30B-A3B, Coder 30B-A3B, Coder-Next), Llama 3.3 70B, Qwen3-235B на двух Spark
- Движки: Ollama, vLLM, llama.cpp, SGLang, TensorRT-LLM
- Также: кластер из двух Spark, дообучение QLoRA через Unsloth, Open WebUI, готовые docker compose
- [`benchmarks/benchmarks.csv`](benchmarks/benchmarks.csv): 138 опубликованных замеров на GB10, у каждого ссылка на источник и уровень надёжности

Все факты проверены **2026-10-20**. У каждого рецепта есть статус: **official** (playbook NVIDIA), **vendor** (документация производителя инструмента), **community-measured** (команда из источника, где опубликован замер) или **untested, from &lt;source&gt;** (мы адаптировали команду, но её никто не проверял). Блоки, которые отличаются от источника, помечены *modified from* или *abridged*. Подробнее в [recipes/README.md](recipes/README.md). Сами рецепты написаны на английском.

---

## Быстрый старт: Ollama, около 2 минут плюс загрузка 14 ГБ

Команда установки взята из [playbook `ollama`](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/ollama/README.md) от NVIDIA. В playbook загружается `qwen2.5:32b`, а мы берём `gpt-oss:20b`, потому что Ollama сама замеряла эту модель на Spark: 58,3 ток/с ([блог Ollama](https://ollama.com/blog/nvidia-spark-performance)). *Modified from playbook: заменён тег модели.*

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama run gpt-oss:20b
```

Веб-чат на порту 8080 ([playbook `open-webui`](https://build.nvidia.com/spark/open-webui)):

```bash
docker run -d -p 8080:8080 --gpus=all \
  -v open-webui:/app/backend/data \
  -v open-webui-ollama:/root/.ollama \
  --name open-webui ghcr.io/open-webui/open-webui:ollama
```

---

## Какой движок под какую модель

Скорость генерации **для одного пользователя** на **одном Spark**, если не указано иное. «Много пользователей» означает суммарную скорость по всем параллельным запросам.

| Модель | Лучший замеренный движок | 1 пользователь, ток/с | Много пользователей, ток/с | Источник (дата) | Рецепты |
|---|---|---|---|---|---|
| **gpt-oss-120b** | vLLM, сборка MXFP4 CUTLASS | **60,7** | 116–153 (8) | [ss121 (2026-08-10)](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | [vLLM](recipes/vllm/gpt-oss-120b.md) · [llama.cpp](recipes/llama.cpp/gpt-oss.md) · [Ollama](recipes/ollama/gpt-oss.md) · [SGLang](recipes/sglang/gpt-oss.md) · [TRT-LLM](recipes/tensorrt-llm/gpt-oss-120b.md) |
| | llama.cpp (без сборки образа) | 58,7 | 262,2 (32) | [llama.cpp benches (2026-02-05)](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | |
| **gpt-oss-20b** | llama.cpp | **83,4** | 473,6 (32) | [llama.cpp benches (2026-02-05)](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | [llama.cpp](recipes/llama.cpp/gpt-oss.md) · [vLLM](recipes/vllm/gpt-oss-20b.md) · [Ollama](recipes/ollama/gpt-oss.md) · [SGLang](recipes/sglang/gpt-oss.md) |
| **Qwen3-30B-A3B** | Ollama, Q4_K_M | **85,2** | 87,5 (8) | [ss121 (2026-08-10)](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | [Ollama](recipes/ollama/qwen3.md) · [vLLM](recipes/vllm/qwen3-30b-a3b.md) |
| | vLLM, NVFP4 | 77,1 | **313,4** (8) | там же | |
| **Qwen3-Coder-30B-A3B** | llama.cpp, Q8_0 | **61,1** | 216,8 (32) | [llama.cpp benches (2026-02-05)](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | [llama.cpp](recipes/llama.cpp/qwen3-coder-30b-a3b.md) · [vLLM](recipes/vllm/qwen3-30b-a3b.md) · [Ollama](recipes/ollama/qwen3.md) |
| **Qwen3-Coder-Next** (80B-A3B) | vLLM, INT4 AutoRound | **74,8** | 289,0 (8) | [ss121 (2026-08-10)](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | [vLLM](recipes/vllm/qwen3-coder-next.md) · [Ollama](recipes/ollama/qwen3.md) |
| **Llama 3.3 70B** (dense) | Ollama, Q4_K_M | **~5** | — | [field-notes (2026-08)](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/hardware-model.md) | [Ollama](recipes/ollama/llama-3.3-70b.md) · [vLLM](recipes/vllm/llama-3.3-70b.md) · [TRT-LLM + draft](recipes/tensorrt-llm/llama-3.3-70b.md) |
| **gpt-oss-120b, 2× Spark** | vLLM 0.17.0 с патчами MXFP4, TP=2 | **81,5** | — | [форум (2026-03-11)](https://forums.developer.nvidia.com/t/vllm-0-17-0-mxfp4-patches-for-dgx-spark-qwen3-5-35b-a3b-70-tok-s-gpt-oss-120b-80-tok-s-tp-2/362824) | [два Spark](recipes/cluster/two-sparks.md) |
| **Qwen3-235B-A22B, 2× Spark** | TensorRT-LLM, NVFP4, TP=2 | **21,65** | — | [форум (2025-12-19)](https://forums.developer.nvidia.com/t/question-on-inference-performance-results-of-qwen3-235b-a22b-on-2x-dgx-spark/355053) | [два Spark](recipes/cluster/two-sparks.md) |

Что следует из этих цифр:

- **MoE быстрее dense.** Генерация упирается в пропускную способность памяти (273 ГБ/с), а MoE-модель читает за токен только активных экспертов. gpt-oss-120b (5,1B активных) работает примерно в 8 раз быстрее Llama 3.3 70B.
- **Один пользователь:** llama.cpp и Ollama стартуют за секунды и вполне конкурентны. Хорошо собранный vLLM в сравнении на одном харнессе выиграл на 3 моделях из 4 ([ss121](https://github.com/sergioamsilva/dgx-spark-field-notes)).
- **Несколько пользователей, API, агенты:** vLLM или SGLang. С настройкой по умолчанию `OLLAMA_NUM_PARALLEL=1` Ollama обрабатывает запросы по очереди: в замере на одном харнессе 8 параллельных запросов дали ту же суммарную скорость, что и один.

---

## Бенчмарки

Подборка из [`benchmarks/benchmarks.csv`](benchmarks/benchmarks.csv). Как читать таблицу, объясняет [методология и оговорки](benchmarks/README.md). Главное правило: не сравнивайте скорость одного запроса с суммарной.

| Модель | Движок | Квант | Генерация, ток/с (польз.) | Prefill, ток/с | Железо | Источник | Дата |
|---|---|---|---|---|---|---|---|
| gpt-oss-120b | llama.cpp b7941 | MXFP4 | 58,7 (1) · 42,8 при контексте 32k | 2 444 (pp2048) | 1× | [ggerganov](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | 2026-02-05 |
| gpt-oss-120b | vLLM, MXFP4 CUTLASS | MXFP4 | 60,7 (1) | — | 1× | [ss121](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | 2026-08-10 |
| gpt-oss-120b | SGLang (`sglang:spark`) | MXFP4 | 50,5 (1) · 203,1 (32) | — | 1× | [LMSYS](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark) | 2025-10-13 |
| gpt-oss-120b | Ollama 0.12.6 | MXFP4 | 41,1 (1) | 1 169 | 1× | [Ollama](https://ollama.com/blog/nvidia-spark-performance) | 2025-10-23 |
| gpt-oss-20b | llama.cpp b7941 | MXFP4 | 83,4 (1) · 473,6 (32) | 4 506 | 1× | [ggerganov](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | 2026-02-05 |
| Qwen3-Coder-30B-A3B | llama.cpp b7941 | Q8_0 | 61,1 (1) · 216,8 (32) | 2 987 | 1× | [ggerganov](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) | 2026-02-05 |
| Qwen3-30B-A3B | vLLM | NVFP4 | 77,1 (1) · 313,4 (8) | — | 1× | [ss121](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | 2026-08-10 |
| Qwen3-Coder-Next | vLLM | INT4 AutoRound | 74,8 (1) · 289,0 (8) | — | 1× | [ss121](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766) | 2026-08-10 |
| Qwen3-14B | TensorRT-LLM | NVFP4 | 22,7 (1) | 5 929 | 1× | [NVIDIA](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks) | 2025-10-24 |
| Llama 3.1 70B | Ollama 0.12.6 | Q4_K_M | 4,4 (1) | 1 911 | 1× | [Ollama](https://ollama.com/blog/nvidia-spark-performance) | 2025-10-23 |
| Nemotron 3 Super 120B-A12B | vLLM nightly + MTP | NVFP4 | 23,45 (1) | — | 1× | [форум](https://forums.developer.nvidia.com/t/nemotron-3-super-120b-a12b-nvfp4-on-single-dgx-spark-23-45-tok-s-spark-arena-com-benhmarks/370070) | 2026-05-13 |
| gpt-oss-120b | vLLM 0.17.0 + патчи, TP=2 | MXFP4 | 81,5 (1) | 6 069 | 2× | [форум](https://forums.developer.nvidia.com/t/vllm-0-17-0-mxfp4-patches-for-dgx-spark-qwen3-5-35b-a3b-70-tok-s-gpt-oss-120b-80-tok-s-tp-2/362824) | 2026-03-11 |
| Qwen3-235B-A22B | TensorRT-LLM 1.2.0rc5, TP=2 | NVFP4 | 21,65 (1) | — | 2× | [форум](https://forums.developer.nvidia.com/t/question-on-inference-performance-results-of-qwen3-235b-a22b-on-2x-dgx-spark/355053) | 2025-12-19 |
| Qwen3.5-397B-A17B | vLLM, TP=4 | INT4 AutoRound | ~37 (1) · ~103 (4) | — | 4× | [spark-vllm-docker](https://github.com/eugr/spark-vllm-docker) | 2026-03-12 |

Лидерборд сообщества с рецептом к каждому результату: [Spark Arena](https://spark-arena.com/leaderboard). Его данные мы пока не импортировали.

---

## Официальные ресурсы NVIDIA

- **Playbooks:** [build.nvidia.com/spark](https://build.nvidia.com/spark), исходники в [NVIDIA/dgx-spark-playbooks](https://github.com/NVIDIA/dgx-spark-playbooks). Чаще всего здесь используются: [ollama](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/ollama/README.md), [open-webui](https://build.nvidia.com/spark/open-webui), [vllm](https://build.nvidia.com/spark/vllm) (14.09.2026 страницу перестроили вокруг NVIDIA Sync, теперь она ведёт на [recipes.vllm.ai](https://recipes.vllm.ai/browse?panel=open&hw=dgx_spark_gb10) и spark-vllm-docker. NGC-команды и матрица протестированных моделей, которые мы цитируем, остались в [legacy README на GitHub](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md)), [llama-cpp](https://build.nvidia.com/spark/llama-cpp), [sglang](https://build.nvidia.com/spark/sglang), [trt-llm](https://build.nvidia.com/spark/trt-llm), [speculative-decoding](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/speculative-decoding/README.md), [connect-two-sparks](https://build.nvidia.com/spark/connect-two-sparks), [nccl](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/playbook-nccl/README.md), [unsloth](https://build.nvidia.com/spark/unsloth), [nemo-fine-tune](https://build.nvidia.com/spark/nemo-fine-tune), [nvfp4-quantization](https://build.nvidia.com/spark/nvfp4-quantization)
- **Характеристики:** [DGX Spark Hardware Overview](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)
- **Версии DGX OS и прошивок:** [DGX Spark Release Notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)
- **Форум:** [NVIDIA Developer Forums, DGX Spark / GB10](https://forums.developer.nvidia.com/)

Ресурсы сообщества: [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker) (образы vLLM и рецепты под модели; на него ссылается и официальный multi-node playbook) · [dgx-spark-field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes) (замеры на одном харнессе, MIT) · [llama.cpp DGX Spark benches](https://github.com/ggml-org/llama.cpp/blob/master/benches/dgx-spark/dgx-spark.md) · [llama-benchy](https://github.com/eugr/llama-benchy) · [Spark Arena](https://spark-arena.com/leaderboard)

---

## Подводные камни

**Железо в одной строке:** у GB10 20 ядер Arm и GPU Blackwell, у них общие 128 ГБ LPDDR5x с пропускной способностью 273 ГБ/с ([спецификация](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)). Compute capability GPU — 12.1 (`sm_121`): llama.cpp выводит «NVIDIA GB10, compute capability 12.1» ([#15396](https://github.com/ggml-org/llama.cpp/discussions/15396)), а [playbook llama-cpp](https://build.nvidia.com/spark/llama-cpp) собирает под `121a-real`. С 2026-10-23 OEM-производители продают ещё и версию на 64 ГБ ([StorageReview](https://www.storagereview.com/news/nvidia-dgx-spark-64gb-october-23-4999-two-units-cluster-to-128gb)). Этот репозиторий рассчитан на 128 ГБ.

### 1. Архитектура aarch64: нужны arm64-образы

GB10 — это ARM64, и образ без варианта `linux/arm64` просто не запустится. Мультиархитектурные образы: NGC (`nvcr.io/nvidia/vllm`, `nvcr.io/nvidia/tensorrt-llm/release`, `nvcr.io/nvidia/pytorch`), а также `vllm/vllm-openai`, `lmsysorg/sglang:latest-cu130`, `ollama/ollama` и `ghcr.io/open-webui/open-webui`. Их манифесты в реестрах мы проверили 2026-10-20. Как проверить любой другой образ: `docker manifest inspect <image> | grep arm64`.

### 2. Unified memory ведёт себя иначе

- CPU и GPU делят одни и те же 128 ГБ. vLLM на узле реально может занять около **117 ГБ** ([форум](https://forums.developer.nvidia.com/t/dgx-spark-multi-node-llm-inference-report-for-qwen3-235b-model/355126)); ОС видит около 121 ГиБ unified memory ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes)). Под дисплей по умолчанию уходит 2 ГБ; с июля 2026 в BIOS можно выставить 4 ГБ ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)).
- `nvidia-smi --query-gpu` показывает `N/A` в полях памяти. Для UMA это нормально, смотрите обычный `nvidia-smi` ([legacy playbook vLLM](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md)).
- Страничный кэш сам GPU не отдаётся. Если модель не грузится, хотя память вроде свободна, выполните `sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'` (так советуют многие playbook NVIDIA). В выводе `free` смотрите на колонку `available`, а не на `used`.
- Держите в памяти только одну большую модель за раз.
- NIM-контейнеры на UMA ограничивают `gpu_memory_utilization` значением 0,50, и модели около 75 ГБ падают по OOM. Берите обычный `vllm/vllm-openai` ([форум](https://forums.developer.nvidia.com/t/nemotron-3-super-120b-a12b-nvfp4-on-single-dgx-spark-23-45-tok-s-spark-arena-com-benhmarks/370070)).

### 3. sm_121 и CUDA 13

- Флаги сборки: llama.cpp `-DCMAKE_CUDA_ARCHITECTURES=121a-real`, NCCL `NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"` (playbook NVIDIA).
- Нужны сборки под CUDA 13: `lmsysorg/sglang:latest-cu130`, колёса PyTorch из `.../whl/cu130`.
- **Без ядер под sm_121 NVFP4 уходит в эмуляцию и выдаёт около 1,1 ток/с.** Те же веса на ядрах FlashInfer CUTLASS дают 77,1 ток/с. Ищите в логе `Using FlashInferCutlassNvFp4LinearKernel` или `[MXFP4] Using backend: CUTLASS` ([форум](https://forums.developer.nvidia.com/t/measured-inference-benchmarks-on-a-single-dgx-spark-same-harness-across-ollama-llama-cpp-and-vllm-notes-data-published/379766), [field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md)).
- vLLM: не используйте `--enforce-eager`. Флаг отключает CUDA graphs; с ним NVFP4 падал с 67 до 23 ток/с (field-notes).

### 4. OOM, swap и зависания

- Если модель не умещается в unified memory, может зависнуть вся машина, а не аварийно завершиться один процесс. Поэтому в кластерном лаунчере сообщества есть `--earlyoom`, который убивает vLLM раньше ([playbook vllm/multi-node](https://build.nvidia.com/spark/vllm/multi-node)). В июле 2026 в DGX OS улучшили обработку OOM на UMA ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html)).
- В большинстве рецептов здесь `--gpu-memory-utilization` 0,7–0,8. Оставляйте запас под ОС.
- **Swap:** официальных рекомендаций NVIDIA по размеру swap для Spark мы не нашли. Не рассчитывайте, что swap поможет запустить модель, которая не влезает в память. Подбирайте модель и KV-кэш под колонку `available`.

### 5. Версии DGX OS, драйвера и прошивок

- Текущий стек Founders Edition ([release notes](https://docs.nvidia.com/dgx/dgx-spark/release-notes.html), прочитано 2026-10-20): **DGX OS 7.5.0**, **драйвер 580.159.03**, **CUDA 13.0.2**, ядро 6.17.
- С июня 2026 OTA-обновления по умолчанию не ставятся при первичной настройке. Обновите свежую машину, прежде чем мерить на ней скорость.
- В феврале 2026 вышло исправление регрессии производительности на нескольких Spark, которая появилась после DGX OS 7.4.0.
- Партнёрские системы на GB10 (ASUS, Dell, HP, Lenovo и другие) могут получать прошивки и обновления ОС позже Founders Edition.
- По данным spark-vllm-docker, **на драйвере 590.x CUDAGraph capture на unified memory GB10 зависает (deadlock)**, поэтому для рецепта Qwen3.5-397B нужен 580.x ([README](https://github.com/eugr/spark-vllm-docker), 2026-03-12).

### 6. Мелочи, на которые легко потерять полдня

- **Медленная загрузка модели через mmap:** отключите mmap в llama.cpp (около 20 ГБ памяти сверху). В текущем master это `-lm none`; в старых сборках, например b7941, на которой сняты опубликованные цифры, — `--no-mmap` / `-mmp 0`. Ещё можно поднять read-ahead NVMe до 8192 КБ ([llama.cpp #16578](https://github.com/ggml-org/llama.cpp/discussions/16578), [field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/loading-performance.md)).
- **Пустой `content` и `finish_reason: length`:** thinking-модель потратила весь `max_tokens` на рассуждение. Увеличьте `max_tokens` (playbook NVIDIA).
- **`PermissionError` на `~/.cache/huggingface` или кэш не удаляется:** контейнеры качают веса от root. Помогает `sudo chown -R "$USER:$USER" ~/.cache/huggingface`.
- **Spark выключается под тяжёлой нагрузкой:** `sudo nvidia-smi -lgc 200,2150` снижает частоту GPU; после перезагрузки настройка сбрасывается ([spark-vllm-docker](https://github.com/eugr/spark-vllm-docker)).
- **Первый запрос медленный:** холодный запрос дал 16 ток/с на модели, которая потом стабильно выдавала около 53 ([field-notes](https://github.com/sergioamsilva/dgx-spark-field-notes/blob/main/docs/vllm-on-spark.md)). Перед замером отбрасывайте прогревочный запрос.

---

## Структура репозитория

```
recipes/            по файлу на пару «движок × модель», статус в шапке каждого
  ollama/ vllm/ llama.cpp/ sglang/ tensorrt-llm/
  cluster/two-sparks.md
  finetune/unsloth-qlora.md
  open-webui/open-webui.md
compose/            vLLM + Open WebUI, Ollama + Open WebUI
benchmarks/         benchmarks.csv и методология
THIRD_PARTY_NOTICES.md, LICENSES/   лицензии цитируемых команд (Apache-2.0, MIT)
```

## Как помочь

Замерили что-то на своём Spark? Откройте issue по шаблону **Benchmark result**. В шаблоне есть поля для модели, движка и его версии, кванта, контекста, точной команды и сырого вывода `llama-bench` / `vllm bench serve`. Подробнее в [CONTRIBUTING.md](CONTRIBUTING.md).

## Лицензия

[MIT](LICENSE). Команды из playbook NVIDIA (Apache-2.0) и репозиториев сообщества остаются под исходными лицензиями, подробнее в [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). У каждой цитируемой команды есть ссылка на источник.

---

<sub>Нужен DGX Spark без покупки? Арендуйте на [SparkHost](https://sparkhost.ru) (Москва).</sub>
