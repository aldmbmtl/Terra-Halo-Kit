# nemotron-super-49b-halo

Pre-configured **Llama-3.3-Nemotron-Super-49B-v1.5-FP8 (ROCM)** inference server for the AMD Ryzen AI Halo — a plain **vLLM** StatefulSet running vLLM with ROCM FP8 support. NVIDIA's text-to-knowledge-graph playbook model, tuned for the Halo's 160GB unified GPU memory.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/Llama-3_3-Nemotron-Super-49B-v1_5-FP8` |
| Image | `rocm/vllm:latest` |
| Engine flags | KV cache fp8 · prefix caching · --quantization fp8 |
| Context | 32768 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 88Gi (container OOM, never host) — footprint ~79GB (weights + KV + overhead), fits Halo's 160GB VRAM |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.65` — ceiling sized to the ~79GB footprint, adjusted for ROCM FP8).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 88Gi.
  Guaranteed reserves the full request — 88Gi + 48Gi fits; two large models cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — currently ungated; covers private repos and future gating.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed).
- GPU accessible via `/dev/dri/card0` direct mount.
- First launch downloads the FP8 checkpoint (~50GB) into the model volume — the 88Gi limit contains the download transient.
- **Hardware validation pending** — FP8 load at 0.65 is the single largest memory consumer in the kit, verified on Halo.

## Memory Note

The Halo's 160GB allocatable VRAM provides significant headroom over the DGX Spark's ~80GB. At 88Gi memory limit with 0.65 util (~79GB ceiling), there is ~81GB of overhead room for KV pool and other operations — compared to ~1GB margin on the Spark. This model fits comfortably one-at-a-time on Halo.