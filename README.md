<p align="center">
  <strong>English</strong> · <a href="README.zh-CN.md">中文</a>
</p>

<h1 align="center">Local AI Stack</h1>

<p align="center">
  <strong>A lightweight LAN-first AI workbench for local text and image models</strong><br>
  Web console, OpenAI-compatible gateway, lazy model workers, and a registry-driven path for adding more local backends.
</p>

<p align="center">
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/Python-3.x-3776AB?style=flat-square&logo=python&logoColor=white" alt="Python 3"></a>
  <a href="#api-surface"><img src="https://img.shields.io/badge/API-OpenAI--compatible-4C78A8?style=flat-square" alt="OpenAI-compatible API"></a>
  <a href="#quick-start"><img src="https://img.shields.io/badge/Mode-LAN%20local-2CA02C?style=flat-square" alt="LAN local mode"></a>
  <a href="#runtime-boundaries"><img src="https://img.shields.io/badge/GPU-lazy%20unload-F28E2B?style=flat-square" alt="Lazy unload for GPU memory"></a>
</p>

<p align="center">
  <a href="#overview">Overview</a> ·
  <a href="#architecture">Architecture</a> ·
  <a href="#quick-start">Quick Start</a> ·
  <a href="#api-surface">API</a> ·
  <a href="#model-registry">Registry</a> ·
  <a href="#operations">Operations</a> ·
  <a href="#community-projects">References</a> ·
  <a href="#license">License</a>
</p>

> [!IMPORTANT]
> This repository is a local deployment scaffold. Model weights under `models/`, generated images under `outputs/`, logs, and virtual environments are intentionally excluded from Git.

<a id="overview"></a>
## Overview

Local AI Stack provides a small local workbench for a single-machine or LAN GPU setup. It currently exposes a Qwen text model and a Qwen image model through direct OpenAI-compatible proxies, plus a lightweight Web UI that also acts as a unified gateway for external clients.

| Goal | Implemented approach | Boundary |
| --- | --- | --- |
| Use local models from a browser or external tools | `web_ui.py` serves the Web UI and `/v1` gateway on port `8080` | Default API key is development-only |
| Avoid keeping large workers resident | Text and image proxies start workers on first request and terminate them after idle timeout | CUDA memory is released by process termination |
| Keep model routing editable | `model_registry.json` defines chat/image models, endpoints, health checks, defaults, and feature tags | Additional backends must expose compatible APIs |
| Preserve a path to mature stacks | Architecture notes map this project to Open WebUI, LiteLLM, vLLM, SGLang, ComfyUI, and LocalAI patterns | References are product/architecture guidance, not copied code |

Current default services:

| Layer | Local URL | LAN URL | Notes |
| --- | --- | --- | --- |
| Web console | `http://127.0.0.1:8080` | `http://<LAN_IP>:8080` | Browser console and Stack status page |
| Unified gateway | `http://127.0.0.1:8080/v1` | `http://<LAN_IP>:8080/v1` | External tools can use this single base URL |
| Text proxy | `http://127.0.0.1:8000/v1` | `http://<LAN_IP>:8000/v1` | Lazy Qwen text worker |
| Image proxy | `http://127.0.0.1:8001/v1` | `http://<LAN_IP>:8001/v1` | Lazy Qwen image worker |

Default API key: `local-dev-key`.

<a id="architecture"></a>
## Architecture

<p align="center">
  <a href="docs/assets/stack-architecture.svg">
    <img src="docs/assets/stack-architecture.svg" alt="Local AI Stack architecture" width="92%">
  </a>
</p>
<p align="center"><em>Figure 1 | The stack separates browser UI, unified gateway, lazy model proxies, workers, and operational status surfaces.</em></p>

<p align="center">
  <a href="docs/assets/console-overview.svg">
    <img src="docs/assets/console-overview.svg" alt="Local AI Stack console overview" width="92%">
  </a>
</p>
<p align="center"><em>Figure 2 | The Web UI provides model selection, chat/image entry points, and stack health visibility.</em></p>

<details>
<summary><strong>Open request-flow diagram</strong></summary>
<br>

<p align="center">
  <a href="docs/assets/request-flow.svg">
    <img src="docs/assets/request-flow.svg" alt="Unified gateway request flow" width="92%">
  </a>
</p>
<p align="center"><em>Figure 3 | External clients can call one gateway, which routes chat and image requests to registered local backends.</em></p>

</details>

The service layering is documented in [docs/STACK.md](docs/STACK.md). Integration notes and local reference snapshots are kept under [integrations/](integrations/) and [references/](references/).

<a id="quick-start"></a>
## Quick Start

Create the local Python environment expected by the scripts:

```bash
git clone https://github.com/rudykon/ai-stack.git
cd ai-stack

python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Download or place model weights under `models/`. The text helper downloads the default Qwen text model from ModelScope:

```bash
.venv/bin/python download_model.py
```

Start the Web UI, unified gateway, text proxy, and image proxy on demand:

```bash
./api.sh
```

Then open:

```text
Web UI:      http://127.0.0.1:8080
Gateway:     http://127.0.0.1:8080/v1
LAN Web UI:  http://<LAN_IP>:8080
LAN Gateway: http://<LAN_IP>:8080/v1
API key:     local-dev-key
```

Keep the terminal open while using the stack. Press `Ctrl+C` to stop the Web UI, both API proxies, and any worker they started.

Quick helpers:

```bash
./api.sh status
./api.sh stop
```

<a id="api-surface"></a>
## API Surface

The unified gateway supports:

```text
GET  /v1/models
POST /v1/chat/completions
POST /v1/images/generations
```

Text example through the gateway:

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local-dev-key' \
  -d '{
    "model": "Qwen/Qwen3.6-27B",
    "messages": [{"role": "user", "content": "Say OK in one word."}],
    "max_tokens": 8,
    "temperature": 0
  }'
```

Image example through the gateway:

```bash
curl http://127.0.0.1:8080/v1/images/generations \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer local-dev-key' \
  -d '{
    "model": "Qwen/Qwen-Image-2512",
    "prompt": "A red circle on a white background",
    "size": "512x512",
    "n": 1,
    "response_format": "b64_json"
  }'
```

Direct proxy endpoints remain available when needed:

```text
Text API:  http://127.0.0.1:8000/v1
Image API: http://127.0.0.1:8001/v1
```

<a id="model-registry"></a>
## Model Registry

Model choices are loaded from [model_registry.json](model_registry.json). The current registry includes:

| Model | Type | Proxy | Features |
| --- | --- | --- | --- |
| `Qwen/Qwen3.6-27B` | Chat | `http://127.0.0.1:8000/v1` | OpenAI chat, streaming, lazy unload, thinking toggle |
| `Qwen/Qwen-Image-2512` | Image | `http://127.0.0.1:8001/v1` | OpenAI images, seed control, lazy unload, local gallery |

Add another OpenAI-compatible backend by extending the registry:

```json
{
  "id": "provider/model-name",
  "label": "Display name",
  "type": "chat",
  "runner": "vllm",
  "base_url": "http://127.0.0.1:8100/v1",
  "health_url": "http://127.0.0.1:8100/health",
  "api_key_env": "API_KEY",
  "default": false,
  "features": ["openai-chat", "streaming"]
}
```

Use `"type": "image"` for image-generation backends that expose the OpenAI Images API surface.

<a id="operations"></a>
## Operations

Health checks:

```bash
curl http://127.0.0.1:8080/api/health
curl http://127.0.0.1:8080/api/stack
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8001/health
```

`worker_running:false` means the corresponding model worker is stopped and GPU memory should be near baseline.

Smoke tests that trigger model loading:

```bash
.venv/bin/python smoke_test_gateway.py
.venv/bin/python smoke_test_qwen_image.py
```

Runtime knobs:

| Variable | Default | Purpose |
| --- | --- | --- |
| `API_KEY` | `local-dev-key` | Gateway and proxy API key |
| `WEB_PORT` | `8080` | Web UI and gateway port |
| `IDLE_UNLOAD_SECONDS` | `300` | Text worker idle unload timeout |
| `IMAGE_IDLE_UNLOAD_SECONDS` | `300` | Image worker idle unload timeout |
| `IMAGE_DTYPE` | `float32` | P40-compatible image precision default |
| `IMAGE_DEVICE_MAP` | `sequential` | P40-compatible image offload default |

<a id="runtime-boundaries"></a>
## Runtime Boundaries

- `models/`, `.venv/`, `logs/`, `outputs/`, `github_token.json`, downloaded archives, and local reference checkouts are ignored by Git.
- The default API key is for local development. Change it before exposing the stack beyond a trusted LAN.
- Qwen-Image uses `IMAGE_DTYPE=float32` and `IMAGE_DEVICE_MAP=sequential` by default on the Tesla P40 path to avoid FP16 black-image failures.
- Worker termination is intentional: on this hardware it is the reliable way to release CUDA memory after idle periods.
- The current text path is a lightweight Transformers worker. vLLM or SGLang can be added later as registered serving backends.

<a id="community-projects"></a>
## Community Projects

This project references community projects for product shape and architecture layering; it does not copy their code.

| Layer | Local implementation | Reference direction |
| --- | --- | --- |
| UI | `web/` + `web_ui.py` | Open WebUI-style local console and model entry points |
| Gateway | `/v1` routes in `web_ui.py` | LiteLLM-style model routing and external tool integration |
| LLM serving | `proxy_qwen36.py` + `serve_qwen36.py` | Future vLLM/SGLang-compatible serving boundary |
| Image workflow | `proxy_qwen_image.py` + `qwen_image_worker.py` | ComfyUI/LocalAI-style local image or multimodal aggregation path |
| Operations | `logs/`, `outputs/`, `/api/stack`, `/health` | Visible status without forcing model load |

Reference snapshots and integration notes are in [docs/STACK.md](docs/STACK.md), [integrations/](integrations/), and [references/](references/).

<a id="repository-map"></a>
## Repository Map

| Path | Purpose |
| --- | --- |
| `api.sh` | On-demand launcher for Web UI, text proxy, and image proxy |
| `web_ui.py` | Web UI backend and unified OpenAI-compatible gateway |
| `web/` | Browser console frontend |
| `model_registry.json` | Model, runner, health, defaults, and feature registry |
| `proxy_qwen36.py`, `serve_qwen36.py` | Lazy text API proxy and internal worker |
| `proxy_qwen_image.py`, `qwen_image_worker.py` | Lazy image API proxy and internal worker |
| `download_model.py`, `download_qwen_image.py` | ModelScope download helpers |
| `smoke_test*.py` | Gateway and model-loading smoke tests |
| `docs/`, `integrations/`, `references/` | Stack notes, integration examples, and reference snapshots |

<a id="license"></a>
## License

No project-level license file is currently included in this repository. Add a license before redistributing or reusing the code beyond your own controlled environment. Third-party models and dependencies retain their own terms.
