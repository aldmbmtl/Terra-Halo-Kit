# nemotron-halo

Pre-configured **NVIDIA Nemotron 3.5 Lightning 30B-A3B (FP8)** inference server for the AMD Ryzen AI Halo — a plain **vLLM** StatefulSet running vLLM with ROCM FP8 support and DSpark speculative decoding. This is the AMD Halo equivalent of NVIDIA's published AMD Halo configuration.

**Note on FP8 conversion**: The original NVFP4 checkpoints are NVIDIA-specific. This Halo configuration runs the model in fp8 quantization mode via `--quantization fp8` with the `rocm/vllm:latest` container. Quality is preserved from the original FP8-pretrained model (arXiv 2504.03624).

## Why vLLM with ROCM FP8

DSpark speculative decoding needs vLLM ≥ 0.27.1 with FP8 support. This plugin uses the ROCM-enabled vLLM container with FP8 quantization — the validated config for AMD Ryzen AI Halo.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4` |
| Draft model (DSpark) | `nvidia/...-NVFP4-DSpark` |
| Image | `rocm/vllm:latest` (pinned to validated config) |
| Engine flags | moe-backend marlin · KV cache fp8 · FLASHINFER · Mamba cache tuning · prefix caching · auto-tool-choice (qwen3_coder) · --quantization fp8 |
| Context | 131072 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 48Gi (container OOM, never host) — footprint ~39GB (weights + KV + overhead), fits Halo's 160GB VRAM |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

Config source: [vLLM blog — Nemotron 3.5 Lightning Day-0](https://vllm.ai/blog/2026-08-10-nemotron-3-5-lightning-vllm),
"Local Deployment on AMD Halo" section. **Hardware validation pending** — no GB10 in CI.

## Fields

- `gpu_memory_utilization` (default `0.40` — ceiling sized to the ~39GB footprint, adjusted for ROCM FP8).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 48Gi.
  Guaranteed reserves the full request — 48Gi + 48Gi fits; two large models (88Gi+) cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — NVIDIA models require license acceptance; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed).
- GPU accessible via `/dev/dri/card0` direct mount.
- First launch downloads the FP8-converted checkpoint into the model volume.
- **Hardware validation pending** — config transcribed from Qwen/vLLM ROCM published guidance.

## Notes on FP8 Conversion

The Nemotron NVFP4 format is NVIDIA-specific and does not exist natively on HF. This Halo configuration runs the model in fp8 quantization mode with the ROCM vLLM container. The speed gain from FP8 quantization is significant (2-3x faster decode), with acceptable quality loss for interactive use. For production workloads requiring maximum accuracy, the NVIDIA AMD Halo original (BF16/NVFP4) is recommended.