# AI Platform: Tantive-III GPU Inference Stack

Local LLM inference and image generation running on the Alliance Fleet's only GPU node.

## Overview

Tantive-III is a dedicated VM on Node-A (Millennium Falcon) with an NVIDIA RTX 4000 Ada (20GB VRAM) passed through via VFIO. The platform runs a full Docker Compose stack: **Ollama** for LLM inference, **OpenWebUI** for browser-based chat, **AnythingLLM** for RAG workflows, and **ComfyUI** for image generation.

| Component | Port | Purpose |
|-----------|------|---------|
| Ollama | 11434 | LLM inference engine |
| OpenWebUI | 3000 | Chat interface (`llm.tima.dev`) |
| AnythingLLM | 3001 | RAG document workflows |
| ComfyUI | 8188 | Image generation (manual launch only) |

## Key Constraints

- **12GB VM RAM ceiling**: PCIe init stalls above this threshold
- **20GB VRAM budget**: Ollama (~12GB) and ComfyUI (~8GB) are mutually exclusive
- **ComfyUI runs with `restart: "no"`**: crash-loop prevention after VFIO lockup incident v2
- **GRUB hardening**: `pcie_aspm=off pci=noaer nmi_watchdog=1` on Node-A

## Documentation

Read in order:

1. **[PROBLEM.md](PROBLEM.md)**  Why the platform exists: idle GPU, API costs, privacy concerns, and the inference gap in the fleet.
2. **[TRADEOFFS.md](TRADEOFFS.md)**  What was gained and sacrificed: cloud vs. local calculus, VFIO lockup incidents, VRAM budgeting, operational overhead, and when to reconsider.
3. **[IMPLEMENTATION.md](IMPLEMENTATION.md)**  How it was built: VFIO setup, VM deployment, docker-compose.yml, model installation, ComfyUI stability rules, and testing.

## Current Models

| Model | Quantization | VRAM (approx) | Use Case |
|-------|-------------|---------------|----------|
| llama3:8b | Q4_0 | ~4.7GB | General inference, BD-1 local backend |
| deepseek-coder-v2:16b | — | Planned | Code generation tasks |

## Related Projects

- **[Security Monitoring](../security-monitoring/)**: Wazuh SIEM with Wazuh-to-Discord alert pipeline
- **[Identity & Access](../identity-access/)**: Authentik SSO zero-trust identity platform