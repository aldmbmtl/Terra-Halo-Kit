# qwen3-8-27b-halo

Pre-configured **Qwen/Qwen3.8-27B-FP8** (FP8) inference server for the AMD Ryzen AI Halo — a plain
**vLLM** StatefulSet running vLLM with ROCM FP8 support. Qwen3.8-27B is a hybrid Mamba+Attention model
(48 Mamba + 16 attention layers) with 256K context, optimized for the Halo's 160GB unified GPU memory
with FP8 weight quantization.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `Qwen/Qwen3.8-27B-FP8` (Apache-2.0) |
| Image | `rocm/vllm:latest` |
| Quantization | FP8 (native ROCM support, ~29GB weights in RAM) |
| Engine flags | KV cache fp8 · prefix caching · reasoning (`qwen3` parser) · auto-tool-choice (`qwen3_coder` parser) |
| Context | 262144 tokens (locked) |
| Concurrency | 5 sequences (locked) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 56Gi (Guaranteed QoS, ~50Gi footprint), fits Halo's 160GB VRAM |
| GPU util | 0.70 (~78GB ceiling adjusted for ROCM) |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.70` — ~78GB ceiling adjusted for ROCM).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.
- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 56Gi.
  Guaranteed reserves the full request — 56Gi fits within the 160GB VRAM allocatable, leaving headroom for KV pool.
- `hf_token` (optional) — currently ungated; covers private repos and future gating.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume (FP8 weights ~60GB).
  `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed).
- GPU accessible via `/dev/dri/card0` direct mount.
- First launch downloads the FP8 checkpoint (~60GB) into the model volume.
- **Hardware validation pending** — config transcribed from Qwen/vLLM ROCM published guidance.

## Notes

- **FP8 tradeoff for coding**: FP8 quantization halves weight precision (BF16 → FP8), trading accuracy for significantly faster decode. For interactive coding assistance (autocomplete, quick questions, refactoring), the speed gain is worth it. For generating large volumes of production-quality code where correctness matters, BF16 precision is safer — FP8 can introduce subtle errors and weaken multi-step reasoning.
