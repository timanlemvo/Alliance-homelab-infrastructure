# Implementation: AI Platform (Tantive-III)

## Architecture Overview

The AI platform runs on Tantive-III a dedicated VM on Node-A (Millennium Falcon, FCM2250) with an NVIDIA RTX 4000 Ada GPU passed through via VFIO. The entire inference stack is containerized in Docker Compose: Ollama for LLM inference, OpenWebUI for browser-based chat, AnythingLLM for RAG workflows, and ComfyUI for image generation.

Tantive-III sits on the Services VLAN (192.168.20.0/24) and is reverse-proxied through NPM with wildcard SSL. All four services are accessible internally; OpenWebUI is exposed externally at `llm.tima.dev`.

**Stack diagram:**

```
Node-A (Millennium Falcon) ─── VFIO Passthrough ───► Tantive-III VM
                                                        │
                                                   Docker Compose
                                                        │
                              ┌──────────┬──────────┬───┴───────┐
                           Ollama   OpenWebUI  AnythingLLM  ComfyUI
                          :11434     :3000       :3001       :8188
```

## Node-A VFIO Setup

GPU passthrough requires IOMMU enabled at the host level and the NVIDIA card isolated from the host kernel before Proxmox boots.

**GRUB configuration** (`/etc/default/grub`):

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer nmi_watchdog=1"
```

Parameter breakdown:

- `intel_iommu=on iommu=pt`: enables IOMMU in passthrough mode for VFIO device assignment.
- `pcie_aspm=off`: disables PCIe Active State Power Management. The RTX 4000 Ada's power state transitions caused silent lockups during VFIO passthrough; disabling ASPM eliminates that failure mode.
- `pci=noaer`: suppresses PCIe Advanced Error Reporting. AER flood events from the GPU during VFIO attach/detach were filling kernel logs and contributing to instability.
- `nmi_watchdog=1`: enables the NMI watchdog. Added after the VFIO lockup incidents so that kernel-level hard lockups trigger automatic recovery instead of requiring a physical power cycle of Node-A.

After updating GRUB, run `update-grub` and reboot. The NVIDIA GPU must be blacklisted from the host via `vfio-pci` module configuration so Proxmox doesn't claim it before Tantive-III starts.

## Tantive-III VM Deployment

The VM is provisioned in Proxmox on Node-A with the following constraints:

- **RAM: 12GB maximum.** Allocating more than 12GB causes a PCIe initialization stall during boot, where the GPU passthrough negotiation fails when the VM's memory footprint exceeds a threshold that interferes with IOMMU mapping. This limit was found empirically after repeated boot failures.
- **CPU:** Allocated from Node-A's host cores, type set to `host` for passthrough compatibility.
- **GPU:** RTX 4000 Ada (20GB VRAM) attached via VFIO PCI passthrough in Proxmox VM hardware config.
- **Network:** Bridged to Services VLAN (192.168.20.0/24), static IP assignment.
- **OS:** Ubuntu Server with NVIDIA driver installed inside the guest. Verify GPU visibility with `nvidia-smi` after first boot.

## Docker Compose Stack

```yaml
version: "3.8"

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - openwebui_data:/app/backend/data
    depends_on:
      - ollama

  anythingllm:
    image: mintplexlabs/anythingllm:latest
    container_name: anythingllm
    restart: unless-stopped
    ports:
      - "3001:3001"
    environment:
      - LLM_PROVIDER=ollama
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - anythingllm_data:/app/server/storage

  comfyui:
    image: yanwk/comfyui-boot:latest
    container_name: comfyui
    restart: "no"
    ports:
      - "8188:8188"
    volumes:
      - comfyui_data:/root
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

volumes:
  ollama_data:
  openwebui_data:
  anythingllm_data:
  comfyui_data:
```

**Critical note on restart policies:** Ollama, OpenWebUI, and AnythingLLM use `restart: unless-stopped` standard for persistent services. ComfyUI uses `restart: "no"`. This is not an oversight. See the stability section below.

## Model Installation

Once the stack is running, pull models into Ollama:

```bash
# Primary inference model
docker exec -it ollama ollama pull llama3:8b

# Verify model loaded
docker exec -it ollama ollama list

# Test inference from CLI
docker exec -it ollama ollama run llama3:8b "Hello, this is a test."
```

The `llama3:8b` model uses Q4_0 quantization roughly 4.7GB of VRAM when loaded. This leaves headroom within the 20GB budget for context window allocation and concurrent requests through OpenWebUI.

For code-specific tasks, `deepseek-coder-v2` is planned as the secondary model:

```bash
docker exec -it ollama ollama pull deepseek-coder-v2:16b
```

**VRAM note:** Loading multiple models simultaneously in Ollama consumes additive VRAM. Ollama keeps the most recently used model warm (~12GB for larger models). Monitor with `nvidia-smi` inside the VM before pulling additional models.

## ComfyUI Stability Rules

ComfyUI is the most dangerous service in the stack. Two VFIO lockup incidents on Node-A established the following non-negotiable rules:

**1. `restart: "no"` is mandatory.** During lockup incident v2, ComfyUI crashed mid-generation. Docker's `unless-stopped` policy restarted it automatically. ComfyUI crashed again. Each restart allocated VRAM without the previous allocation being properly released. The crash-loop exhausted all 20GB of VRAM within seconds, the GPU driver stalled on PCIe, and Node-A hard-locked. Setting `restart: "no"` means a crash stays a crash, it doesn't cascade into a host-level lockup.

**2. VRAM budget: 20GB total, zero-sum.** Ollama warm-loads models at ~12GB. ComfyUI needs ~8GB for Stable Diffusion workflows. These cannot run simultaneously. GPU mode switching scripts handle the transition: stop Ollama, verify VRAM is clear via `nvidia-smi`, then start ComfyUI (or vice versa).

**3. Manual launch only.** ComfyUI must be started deliberately after confirming VRAM availability:

```bash
# Verify VRAM is clear
nvidia-smi

# Only then start ComfyUI
docker compose up -d comfyui

# Monitor VRAM during use
watch -n 2 nvidia-smi
```

**4. VM RAM stays at 12GB.** Increasing Tantive-III's RAM above 12GB causes PCIe initialization stalls during boot. The GPU either fails to attach or the VM hangs entirely. This is an IOMMU mapping constraint, not a Proxmox bug.

## Testing and Access

Once deployed, verify the stack end-to-end:

1. **Ollama API**: `curl http://192.168.20.X:11434/api/tags` should return the model list.
2. **OpenWebUI**: Browse to `http://192.168.20.X:3000`, create an admin account on first launch, and send a test prompt. Confirm the model selector shows `llama3:8b`.
3. **External access**: Navigate to `https://llm.tima.dev` (routed through NPM with wildcard SSL). Verify SSL handshake and login.
4. **AnythingLLM**: Access at `:3001`, configure Ollama as the LLM provider, and test a basic RAG query.
5. **ComfyUI**: Manually start per the rules above, access at `:8188`, and run a basic txt2img generation. Monitor `nvidia-smi` throughout.

If inference works through OpenWebUI and `nvidia-smi` shows VRAM allocation on the Ada, the platform is operational.
