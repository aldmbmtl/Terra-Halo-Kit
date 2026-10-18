# AMD Ryzen AI Halo Kit

A series of [Terra](https://juno-fx.github.io/Orion-Documentation/) plugins optimized for small devices
like the AMD Ryzen AI Halo. Each plugin is a Helm chart that Terra installs into Kubernetes clusters via ArgoCD.

This repository is a Terra plugin **Source** — add it in the Terra UI like any other source, alongside
or instead of the [official plugin catalog](https://github.com/juno-fx/Terra-Official-Plugins).
Plugins use distinct `-spark` resource IDs so they coexist cleanly with their official counterparts.
The one deliberate exception is the per-model `-unify` companion set (see [AGENTS.md](AGENTS.md)).

---

## Plugin Catalog

| Plugin | Type | Category | Description | Docs |
|--------|------|----------|-------------|------|
| Nemotron 3.5 Lightning | Workload Template | AI | 30B-A3B NVFP4 on plain vLLM (pinned v0.27.1) with DSpark speculative decoding — AMD Ryzen AI Halo configuration | [README](plugins/nemotron-halo/README.md) |
| Nemotron Coding (Halo) | Workload Template | AI | Nemotron 3.5 Lightning 30B-A3B NVFP4 on plain vLLM with DSpark speculative decoding and 512K context window for coding — 4× ctx expansion, Guaranteed QoS preserved. | [README](plugins/nemotron-coding-halo/README.md) |
| Nemotron 3 Nano | Workload Template | AI | 30B-A3B NVFP4 (QAD) on plain vLLM — AMD Ryzen AI Halo playbook model | [README](plugins/nemotron-nano-halo/README.md) |
| Qwen3 30B-A3B (Halo) | Workload Template | AI | Qwen3-30B-A3B-Instruct-2507 FP8 on plain vLLM with ROCM, 128K context | [README](plugins/qwen3-halo/README.md) |
| Qwen3.6 35B-A3B (Halo) | Workload Template | AI | Qwen3.6-35B-A3B-FP8 on plain vLLM with ROCM, 128K context, 4 seqs | [README](plugins/qwen3-6-halo/README.md) |
| Nemotron Super 49B | Workload Template | AI | Llama-3_3-Nemotron-Super-49B-v1_5-FP8 on plain vLLM with ROCM | [README](plugins/nemotron-super-49b-halo/README.md) |
| Open WebUI | Workload Template | AI | Chat UI pointed at any kit model workload — CPU-only companion, co-schedules with one model | [README](plugins/open-webui-spark/README.md) |
| ComfyUI | Workload Template | AI | Node-graph image generation (SDXL/Flux) — GPU-exclusive workload, AMD Halo-native container, models managed via the UI | [README](plugins/comfy-ui-spark/README.md) |
| Open WebUI | Workload Template | AI | Chat UI pointed at any kit model workload — CPU-only companion, co-schedules with one model | [README](plugins/open-webui-spark/README.md) |
| Qwen3.8-27B (Halo) | Workload Template | AI | Qwen3.8-27B-FP8 on plain vLLM with ROCM, 256K context, 56Gi | [README](plugins/qwen3-8-27b-halo/README.md) |
| Nemotron-H 56B Base | Workload Template | AI | 56B hybrid Mamba-2 + 10 attention layers (Nemotron-H) on plain vLLM with load-time dynamic FP8, 8K context — research-only license | [README](plugins/nemotron-h-56b-halo/README.md) |
| Nemotron 3.5 Lightning Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running  model | [README](plugins/nemotron-unify/README.md) |
| Nemotron Coding Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running `nemotron-coding-halo` model | [README](plugins/nemotron-coding-unify/README.md) |
| Nemotron 3 Nano Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running  model | [README](plugins/nemotron-nano-unify/README.md) |
| Qwen3 30B-A3B Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running `qwen3-halo` model | [README](plugins/qwen3-unify/README.md) |
| Qwen3.6 35B-A3B Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running `qwen3-6-halo` model | [README](plugins/qwen3-6-unify/README.md) |
| Nemotron Super 49B Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running  model | [README](plugins/nemotron-super-49b-unify/README.md) |
| Qwen3.8-27B Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running `qwen3-8-27b-halo` model | [README](plugins/qwen3-8-unify/README.md) |
| Nemotron-H 56B Unify | Workload Template (no pod) | AI | Service + optional Ingress/NodePort/LoadBalancer front-end for the running  model | [README](plugins/nemotron-h-56b-unify/README.md) |

Bundles: [AMD Halo LLM Stack](bundles/halo-llm.yaml) — all **eight** model workloads in one install ·
[AMD Halo AI Lab](bundles/halo-ai-lab.yaml) — Open WebUI + ComfyUI.
The eight `-unify` front-ends are deliberately **not** bundled: launch one manually after its model.

Every **model** workload is a plain-vLLM StatefulSet with a footprint-derived unified-memory budget
(memory request = limit per model: 48Gi Lightning/Nano (NVIDIA), 56Gi Qwen3.8-27B, 72Gi Qwen3.6/Nemotron-H-56B, 88Gi Qwen3/Super-49B; util 0.40–0.70),
served under the platform `Halo-native ingress` prefix via an in-pod nginx prefix-stripping sidecar. Each
model sidecar also carries a generic `` route, which the per-model `-unify` companions
front (Service + optional Ingress — no pod of their own).

Eleven **non-model companion workloads** (Open WebUI, ComfyUI and the eight `-unify`
front-ends) follow the same ingress rule; the `-unify` front-ends add the sanctioned extra path
segment (`<name>`) and delegate their prefix strip to the target model pod's sidecar.
Launch a companion in the same project as its model, with the model launched under its plugin id —
otherwise the companion's label selector matches nothing. Model workloads are GPU-exclusive (one at
a time); the CPU-only companions co-schedule alongside any one model (48Gi+4Gi and 88Gi+4Gi both fit
the 117.5Gi allocatable node).

---

## Adding as a Terra Source

1. Push this repository to GitHub
2. In the Terra UI, add a new Source pointing at this repository URL
3. Terra scans `plugins/*/terra.yaml` and loads every plugin automatically

---

## Development

```bash
# 1. Enter the dev environment (required for all make targets)
devbox shell

# 2. Create a new plugin (interactive — prompts for type)
make new-plugin

# 3. Edit your plugin files

# 4. If your plugin has a scripts/ directory, package it
make package <plugin-name>

# 5. Verify nothing is stale
make verify
make lint
```

## Key Commands

| Command | Description |
|---------|-------------|
| `make new-plugin` | Interactive scaffolding for namespaced / cluster / workload plugins |
| `make package <name>` | **Required** after any `scripts/` change — bundles scripts into a ConfigMap |
| `make verify` | Checks all plugins have up-to-date packages (runs in CI on every push) |
| `make check-size <name>` | Checks packaged size against the 1MiB Kubernetes ConfigMap limit |
| `make watch <name>` | Auto-repackages when `scripts/` changes |
| `make lint` | Helm lint all plugins |
| `make test <name>` | Deploys plugin to a local Kind cluster with ArgoCD |
| `make down` | Destroys the local Kind cluster |

## The Packaging Rule

> **If you change anything in `scripts/` or `scripts/chart/`, you MUST run `make package <plugin-name>`.**
> If you skip repackaging, the old version deploys silently with no error. `make verify` catches this.

See [AGENTS.md](AGENTS.md) for the full rules. For authoritative upstream guidance, read the
[Terra-Official-Plugins AGENTS.md](https://github.com/juno-fx/Terra-Official-Plugins/blob/main/AGENTS.md).

## License

MIT — see [LICENSE](LICENSE).
