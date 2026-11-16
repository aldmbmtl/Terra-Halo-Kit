# nemotron-coding-halo

Pre-configured **NVIDIA Nemotron 3.5 Lightning 30B-A3B (BF16)** inference server for the AMD Ryzen AI Halo — a plain **vLLM** StatefulSet on the ROCm build of vLLM, with a 512K context window for coding.

This is the general Nemotron 3.5 Lightning model with coding-oriented settings (512K context, one sequence, `qwen3_coder` tool parser). There is no separate coding checkpoint.

## Why BF16 on the Halo

The Halo's GPU is a Radeon 8060S (`gfx1151`, Strix Halo). The ROCm build of vLLM has no FP8 linear kernel for it: the native ROCm kernel requires CDNA3+ (`gfx942`/`gfx950`) or RDNA4 (`gfx12x`), and the AITER kernels are gated to CDNA3+ regardless of `VLLM_ROCM_USE_AITER`. That rules out:

- the **NVFP4** checkpoint — its Mamba projection layers are FP8-quantized (fails at load with `Failed to find a kernel that can implement the ScaledMM linear layer`);
- load-time **`--quantization fp8`** — it needs the same missing kernel.

The full-precision BF16 checkpoint needs no quantized kernel, so it is the one that loads on this hardware.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16` (61.3 GiB on disk) |
| Image | `rocm/vllm:latest` |
| Engine flags | kv cache fp8 · Mamba backend `triton` · Mamba SSM cache float16 · `--mamba-cache-mode align` · prefix caching · `--reasoning-parser nemotron_v3` · auto-tool-choice (`qwen3_coder`) |
| Speculative decoding | None. DSpark is not configured: the only published draft model is NVFP4 (W4A16) and is untested on ROCm |
| Context | **524288 tokens (512K)** — the model supports up to 1M |
| Concurrency | **1 sequence** — sufficient for single long coding completions |
| GPU | 1 × AMD Radeon 8060S iGPU (hostPath `/dev/kfd` + `/dev/dri`, privileged), no runtime class |
| Node | `nodeSelector: juno-innovations.com/amd-halo: "true"` |
| Memory | 76Gi request = limit (Guaranteed QoS) |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix before proxying to vLLM (`:8000`).

### Flags that differ from NVIDIA's published recipe

NVIDIA's model card recipe uses `--mamba-backend flashinfer`, `--enable-mamba-cache-stochastic-rounding` and `--mamba-cache-philox-rounds 5`. None of these work on ROCm: FlashInfer is CUDA-only and is not in the image, and vLLM rejects stochastic rounding off CUDA. `--mamba-ssm-cache-dtype float16` is kept without stochastic rounding; if long-context output degrades, drop that flag so the model's own `float32` default applies.

## Memory footprint

| Component | Size |
|-----------|------|
| Weights (BF16) | ~66 GB |
| KV pool (6 attention layers × 2 KV heads × 128 head dim × 524288 tokens × 1 seq, fp8) | ~1.6 GB |
| Activations + Mamba state | ~5 GB |
| GPU ceiling (`gpu_memory_utilization` 0.72 × ~101 GB) | ~73 GB |
| Container `memory` (request = limit) | 76Gi (~81.6 GB) |

The Halo has unified memory: GPU allocations come out of system RAM (~125Gi total). 76Gi is sized so the GPU ceiling plus the process's own RAM sits below the limit, and so the pod still schedules next to the node's other workloads. Whether GPU allocations are charged to the container's memory cgroup on this platform is not verified.

## Fields

- `gpu_memory_utilization` (default `0.72` — ~73 GB ceiling on the ~101 GB device).
- `served_model_name` (default `model`) — the model string clients send in API requests (the OpenAI-compatible `model` field). Standardized across all kit workloads.
- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 76Gi. One model at a time on the node.
- `hf_token` (optional) — the checkpoint is not gated today; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume.

## Prerequisites

- AMD GPU drivers on the host with `/dev/kfd` and `/dev/dri` present (no nvidia-gpu-operator needed).
- Node labelled `juno-innovations.com/amd-halo=true`.
- **Slow first boot**: downloads ~61 GiB of weights into the model volume, then loads them. Restarts resume from the cached download. The startup probe allows 50 minutes (threshold 300).

## License

The model is released under the **OpenMDW-1.1** license (not Apache-2.0). NVIDIA positions the BF16 release primarily for customization and recommends the NVFP4 release for deployment; BF16 is used here because NVFP4 cannot load on this GPU.

## Validation status

Hardware validation pending — this is an assembled config, not an NVIDIA-published recipe. Launch ladder: node stays Ready, pod 2/2, `/v1/models` answers `id: "model"` with `max_model_len` 524288, peak container memory < 76Gi.

## Known failure signatures

- `Failed to find a kernel that can implement the ScaledMM linear layer` = an FP8 path was re-introduced (NVFP4/FP8 checkpoint or `--quantization fp8`). Not supported on `gfx1151`. If it appears with the BF16 checkpoint, the suspect is `--kv-cache-dtype fp8`; change it to `auto`.
- `Stochastic rounding for Mamba cache is only supported on NVIDIA CUDA platforms` = `--enable-mamba-cache-stochastic-rounding` was re-added.
- `Invalid repository ID or local directory specified: ''` = a `--speculative_config.model` arg rendered empty.
- **exit 2 fast** = flag removed on `:latest` drift.
- CrashLoopBackOff 401 = `HF_TOKEN` wiring (keep the secret key named `HF_TOKEN`).
