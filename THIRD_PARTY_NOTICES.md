# Third-party notices

This repository's own text, CSV and compose files are licensed under the [MIT License](LICENSE). Some recipes quote shell commands, config snippets and YAML from the projects below. Those excerpts remain under their original licenses, listed here.

Every quoted block links to its source. Where we changed a block (a different model, an added flag, `docker run` turned into compose), it is marked *modified from &lt;source&gt;* or *abridged*, and the change is described next to it. This satisfies Apache-2.0 §4(b), which requires modified files to carry prominent notices of change.

Measured numbers (tokens/s, dates, versions) are facts reported by their sources. They are cited, not licensed.

---

## NVIDIA DGX Spark playbooks

- Source: <https://github.com/NVIDIA/dgx-spark-playbooks> (also published at <https://build.nvidia.com/spark>)
- License: **Apache License 2.0**. Full text in [LICENSES/Apache-2.0.txt](LICENSES/Apache-2.0.txt), copied from the repository's `LICENSE`. Also at <https://www.apache.org/licenses/LICENSE-2.0>.
- Copyright: "Copyright 2025 NVIDIA Corporation" (from the repository's LICENSE)
- Used in: almost all recipes marked **official**, and both compose files (modified)

## eugr/spark-vllm-docker

- Source: <https://github.com/eugr/spark-vllm-docker>
- License: **MIT**
- Used in: `recipes/vllm/gpt-oss-120b.md`, `recipes/vllm/qwen3-coder-next.md`, `recipes/cluster/two-sparks.md` (recipe YAML commands, script invocations)

```
MIT License

Copyright (c) 2026 Eugene Rakhmatulin

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## sergioamsilva/dgx-spark-field-notes

- Source: <https://github.com/sergioamsilva/dgx-spark-field-notes>
- License: **MIT**
- Used in: vLLM and Ollama recipes (minimal `vllm serve` invocation, Ollama environment variables, image tagging steps), and benchmark rows

```
MIT License

Copyright (c) 2026 Sérgio Silva

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## llama.cpp

- Source: <https://github.com/ggml-org/llama.cpp>. The `llama-bench` / `llama-batched-bench` / `llama-server` commands come from `benches/dgx-spark/dgx-spark.md` and discussion [#16578](https://github.com/ggml-org/llama.cpp/discussions/16578)
- License: **MIT**
- Used in: `recipes/llama.cpp/*`, `CONTRIBUTING.md`

```
MIT License

Copyright (c) 2023-2026 The ggml authors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## Unsloth

- Source: the [Unsloth DGX Spark guide](https://unsloth.ai/docs/blog/fine-tuning-llms-with-nvidia-dgx-spark-and-unsloth). `Dockerfile_DGX_Spark` and the notebooks live in [unslothai/notebooks](https://github.com/unslothai/notebooks) (LGPL-3.0)
- Used in: `recipes/finetune/unsloth-qlora.md`, Path B. We quote only the short `wget` / `docker build` / `docker run` / `jupyter` commands from the guide. The Dockerfile and notebooks are downloaded from their source at run time and are not redistributed here.
