# nemotron-h-56b-halo

Pre-configured **Nemotron-H-56B-Base-8K (FP8 dynamic, Mamba-2 hybrid)** inference server for the AMD Ryzen AI Halo — a plain **vLLM** StatefulSet running vLLM with ROCM FP8 dynamic quantization and Mamba-2 hybrid support. This is the AMD Halo equivalent of the research-only Nemotron-H-56B model.

**⚠️ License**: this model ships under the **NVIDIA Internal Scientific Research and Development Model License** — **"for research and development only"** (not the kit baseline Apache-2.0). Confirm your entitlement before deploying.

> **ℹ️ Base model**: completion-only — **no chat template** (no 56B instruct variant exists). Call OpenAI `/v1/completions` for raw continuations; chat/UIs will show raw text.

## Why dynamic FP8 on ROCM

The HF checkpoint is bf16 — **104.9Gi on disk**, which alone exceeds the ~100GB usable unified-memory budget. No FP8/INT4 checkpoint exists anywhere on HF (verified author-agnostic searches). Dynamic FP8 halves the in-RAM weight footprint to **~52.5Gi** at load time. Quality-safe: the paper (arXiv 2504.03624) reports **56B-Base was itself fully pre-trained with an FP8 recipe**. Ungated — no token needed (still shipped per kit convention).

**Critical**: This workload requires `--mamba-cache-mode align + --enable-chunked-prefill`. These flags are mandatory for Mamba-2 hybrid models and must not be dropped.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/Nemotron-H-56B-Base-8K` (ungated, research-only) |
| Image | `rocm/vllm:latest` |
| Engine flags | `--quantization fp8` (dynamic) · `--kv-cache-dtype fp8` · prefix caching · `--mamba-cache-mode align` · `--enable-chunked-prefill` · trust-remote-code |
| Context | 8192 tokens (locked — native max) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × AMD Radeon iGPU (via `/dev/dri/card0` direct mount), no runtime class needed |
| Memory limit | 72Gi (container OOM, never host) — footprint ~63Gi (52.5Gi FP8 weights + KV + activations + runtime), fits Halo's 160GB VRAM |
| util ceiling | 0.55 → ~70GB (below the 72Gi cgroup, ~7GB margin) |
| Storage | 120Gi default — bf16 cache is 104.9Gi on disk (quant happens in-RAM at load) |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`), 600s timeouts.

## Fields

- `gpu_memory_utilization` (default `0.55` — ceiling sized to the ~70Gi footprint, adjusted for ROCM FP8).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.
- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 72Gi.
  Guaranteed reserves the full request — 72Gi + 48Gi and 72Gi + 88Gi exceed 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — ungated model; covers private repos / future gating.
- `storage_class` (required) / `storage_size` (default 120Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- ROCM 10.0+ installed with AMD GPU drivers (no nvidia-gpu-operator needed).
- GPU accessible via `/dev/dri/card0` direct mount.
- **Slow first boot**: download ~105Gi weights then FP8-quantize 56B params at load time. The startup probe allows **50 minutes** (300 × 10s); restarts resume from the cached download.
- **Hardware validation pending** — this is an **assembled** config (vLLM Nemotron-H support ≥ 0.21 + the model's FP8-pretrained claim), not an NVIDIA-published Spark playbook. First launch ladder: node stays Ready, `2/2` pods, `/v1/models` answers `id: "model"`, peak container memory < 72Gi.

## Critical: Mamba-2 Flags (DO NOT DROP)

This workload **requires** both of these flags:
- `--mamba-cache-mode align` — enables Mamba-2 cache format
- `--enable-chunked-prefill` — enables chunked prefill for Mamba-2

Dropping either flag will cause the model to fail to load. These flags are mandatory for all Mamba-2 hybrid models in the kit.

## Known failure signatures

- **exit 2 fast** = flag removed on `:latest` drift (first suspect: `--mamba-cache-mode` / `--enable-chunked-prefill` / `--quantization`). **Do not drop `--quantization fp8` to recover** — bf16 (~105Gi) does not fit; repackage with the updated image instead.
- CrashLoopBackOff 401 = `HF_TOKEN` wiring (not gated today, but keep the secret key named `HF_TOKEN`).
- **Startup hang** = missing `--mamba-cache-mode align` or `--enable-chunked-prefill` flags. Verify statefulset args.