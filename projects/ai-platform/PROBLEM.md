# Problem Statement: AI Platform (Tantive-III)

## Initial State

Tantive-III, a dedicated VM on Node-A (Millennium Falcon), had an NVIDIA RTX 4000 Ada with 20GB of VRAM passed through via VFIO, but no workloads consuming it. The GPU sat idle while every AI interaction in the Alliance Fleet routed through external cloud APIs. The most powerful piece of hardware in the cluster had nothing to do.

## The Pain

Every Claude and GPT API call cost money. Token-based pricing added up fast across BD-1's Discord interactions, n8n workflow automations, and ad-hoc queries. Beyond cost, round-trip latency to external endpoints introduced delays that broke the responsiveness expected from a homelab assistant. Worst of all, every prompt and response transited third-party infrastructure fleet data, server configs, and personal context leaving the network with every request.

## Specific Gaps

There was no local LLM inference capability anywhere in the fleet. No Ollama instance for running open-weight models on-device. No image generation pipeline ComfyUI or otherwise leveraging the Ada's compute. The 20GB of VRAM existed purely as a spec sheet number.

## Why It Mattered

BD-1 needed a local inference backend to reduce API dependency and enable private, low-latency responses for fleet operations. ComfyUI would unlock on-device image generation for creative workflows. Without a local AI platform, the RTX 4000 Ada was an expensive heater and the fleet's most critical capability gap.
