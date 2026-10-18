# nemotron-nano-halo

Pre-configured **NVIDIA Nemotron 3 Nano 30B-A3B (NVFP4 QAD, FP8)** inference server for the AMD Ryzen AI Halo — a plain **vLLM** StatefulSet running vLLM with ROCM FP8 support and DSpark speculative decoding. This is the AMD Halo equivalent of the official NVIDIA Spark playbook for Nemotron 3 Nano.

**Note on FP8 conversion**: The original NVFP4 checkpoints are NVIDIA-specific. This Halo configuration runs the model in fp8 quantization mode via `--quantization fp8` with the `rocm/vllm:latest` container. Quality is preserved from the original FP8-pretrained model where applicable.

## Why vLLM with ROCM FP8

DSpark speculative decoding needs vLLM ≥ 0.27.1 with FP8 support. This plugin uses the ROCM-enabled vLLM container with FP8 quantization — the validated config for AMD Ryzen AI Halo running the Nan o 30B-A3B QAD model.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4` |
| Image | `rocm/vllm:latest` (pinned to validated config) |
| Engine flags | FLASHINFER (`VLLM_ATTENTION_BACKEND`) · KV cache fp8 · prefix caching · auto-tool-choice (qwen3_coder) · deepseek_r1 reasoning parser · --quantization fp8 |
| Context | 131072 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 48Gi (container OOM, never host) — footprint ~38GB (weights + KV + overhead), fits Halo's 160GB VRAM |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.40` — ceiling sized to the ~38GB footprint, adjusted for ROCM FP8).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 48Gi.
  Guaranteed reserves the full request — 48Gi + 48Gi fits; two large models (88Gi+) cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — NVIDIA models require license acceptance; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 50Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed).
- GPU accessible via `/dev/dri/card0` direct mount.
- First launch downloads the FP8-converted checkpoint into the model volume.
- **Hardware validation pending** — config transcribed from Qwen/vLLM ROCM published guidance.

## Notes on FP8 Conversion

The Nemotron NVFP4 QAD format is NVIDIA-specific and does not exist natively on HF. This Halo configuration runs the model in fp8 quantization mode with the ROCM vLLM container. The speed gain from FP8 quantization is significant (2-3x faster decode), with acceptable quality loss for interactive use. For production workloads requiring maximum accuracy, the NVIDIA DGX Spark original (BF16/NVFP4 QAD) is recommended.