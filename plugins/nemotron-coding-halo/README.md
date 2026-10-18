# nemotron-coding-halo

Pre-configured **NVIDIA Nemotron 3.5 Lightning 30B-A3B (FP8, Halo)** inference server for the AMD Ryzen AI Halo — a plain **vLLM** StatefulSet running vLLM with ROCM FP8 and DSpark speculative decoding, the AMD Halo equivalent of the DGX Spark configuration with a 512K context window for coding.

## Why dynamic FP8 on ROCM

The original NVFP4 checkpoint is served with load-time dynamic FP8 quantization (`--quantization fp8`), which halves the in-RAM weight footprint from ~52.5Gi to ~26Gi while preserving the 512K context window capability. Quality-safe: the model was FP8-pretrained. No custom image build needed — uses `rocm/vllm:latest`.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4` |
| Draft model (DSpark) | `nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4-DSpark` |
| Image | `rocm/vllm:latest` |
| Engine flags | kv cache fp8 · FLASHINFER · Mamba cache tuning · prefix caching · auto-tool-choice (qwen3_coder) · --quantization fp8 |
| Context | **524288 tokens (512K — 4× original 131072; KV pool identical to original, footprint ~39Gi unchanged; gpu_memory_utilization: 0.40)** |
| Concurrency | **1 sequence** (KV pool 524288 × 1 = 131072 × 4; sufficient for single long coding completions) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 48Gi (container OOM, never host) — footprint ~39Gi (weights + KV + overhead), fits Halo's 160GB VRAM |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix before proxying to vLLM (`:8000`).

Larger Context for Coding: This plugin provides 512K token context windows for code completions, entire codebases, and multi-file refactoring. The KV pool (524288 × 1 = 524288) is identical to the original 131072 × 4, so memory footprint and gpu_memory_utilization (0.40) are unchanged. Only 1 concurrent sequence — sufficient for single long coding completions (one at a time).

Hardware validation pending — this is an assembled config (vLLM ROCM support + the model's FP8-pretrained claim), not NVIDIA-published Spark playbook. First launch ladder: node stays Ready, 2/2 pods, `/v1/models` answers `id: "model"`, peak container memory < 48Gi.

## Fields

- `gpu_memory_utilization` (default `0.40` — ceiling sized to the ~39Gi footprint, adjusted for ROCM FP8). Enables 512K ctx with seq=1.
- `served_model_name` (default `model`) — the model string clients send in API requests (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 48Gi. Guaranteed reserves the full request — 48Gi + 48Gi fits; two large models (88Gi+) cannot co-schedule on 117.5Gi, so the node fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed). GPU accessible via `/dev/dri/card0` direct mount.
- First launch downloads the NVFP4 + DSpark checkpoints (~18GB) into the model volume — the 48Gi limit contains the download transient.
- **Slow first boot**: download ~18GB weights then FP8-quantize at load time with DSpark speculative decoding. Restarts resume from cached download.

## Known failure signatures

- **exit 2 fast** = flag removed on `:latest` drift (first suspect: `--quantization fp8` / `--mamba-cache-mode` / `--enable-auto-tool-choice`). Do not drop `--quantization fp8` to recover — bf16 (~39Gi) does not fit; repackage with the updated image instead.
- CrashLoopBackOff 401 = `HF_TOKEN` wiring (not gated today, but keep the secret key named `HF_TOKEN`).
- **Startup hang** = missing `--mamba-cache-mode align` or `--reasoning-parser nemotron_v3` flags. Verify statefulset args.
