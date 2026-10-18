# qwen3-6-halo

Pre-configured **Qwen/Qwen3.6-35B-A3B-FP8** (FP8) inference server for the AMD Ryzen AI Halo — a plain
**vLLM** StatefulSet running vLLM with ROCM FP8 support. Qwen3 is the most widely used open LLM family
(33.9M+ downloads on Ollama); the 35B-A3B MoE activates only 3B parameters per token, making it very fast
on the Halo's 160GB unified GPU memory.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `Qwen/Qwen3.6-35B-A3B-FP8` (Apache-2.0) |
| Image | `rocm/vllm:latest` |
| Engine flags | KV cache fp8 · prefix caching · reasoning (`deepseek_r1` parser) · auto-tool-choice (hermes parser) |
| Context | 131072 tokens (locked; model supports 262144) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 72Gi (container OOM, never host) — footprint ~64GB (35GB FP8 + 21.5GB KV + overhead), fits Halo's 160GB VRAM |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.55` — ~70GB ceiling adjusted for ROCM).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 72Gi.
  Guaranteed reserves the full request — 72Gi + 48Gi fits; two large models cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
  *Note: On Halo, 72Gi fits within the 160GB VRAM allocatable, leaving headroom for KV pool.*
- `hf_token` (optional) — currently ungated; covers private repos and future gating.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume (FP8 weights ~60GB).
  `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed).
- GPU accessible via `/dev/dri/card0` direct mount.
- First launch downloads the FP8 checkpoint (~60GB) into the model volume.
- **Hardware validation pending** — config transcribed from Qwen/vLLM ROCM published guidance.
