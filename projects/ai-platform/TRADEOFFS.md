# Trade-offs: Local GPU Inference on Tantive-III

## Why Local AI Over Cloud APIs

The decision to run inference locally on the RTX 4000 Ada came down to three factors: cost, privacy, and latency.

Cloud API pricing is token-based and unpredictable. BD-1's Discord interactions, n8n workflow automations, and ad-hoc queries generate thousands of API calls monthly. At scale, OpenAI and Anthropic pricing compounds, especially when workflows chain multiple calls. Local inference has a fixed cost: the hardware already exists.

Privacy was non-negotiable. Every cloud API call ships prompt context server configurations, fleet topology, operational data to third-party infrastructure. Local Ollama means fleet data never leaves the network.

Latency matters for interactive use. Cloud round-trips add 200-500ms of network overhead before a single token generates. Local inference on the Ada eliminates that entirely. For BD-1's Discord responses and n8n webhook-triggered workflows, the difference between "instant" and "noticeable delay" is the difference between a useful tool and an annoying one.

## What Was Sacrificed

Self-hosting is not free. The trade-offs are real and ongoing.

Model size hits a hard ceiling. The RTX 4000 Ada has 20GB of VRAM enough for 7B-13B parameter models comfortably, but 70B+ models either don't fit or run partially offloaded to CPU at unusable speeds. Cloud APIs serve 100B+ parameter models with no VRAM concern.

Inference speed is GPU-bottlenecked. The Ada is a workstation card, not a datacenter accelerator. Token generation for larger quantized models is measurably slower than cloud endpoints backed by A100/H100 clusters.

Model availability lags behind. New open-weight models need quantization, testing, and VRAM profiling before deployment. Cloud APIs just update an endpoint.

## The VFIO/GPU Stability Issues

Two separate lockup incidents shaped the platform's stability posture. Both presented identically Node-A hard-locked, no SSH, no console, requiring a physical power cycle but the root causes were completely different.

**VFIO Lockup Incident v1** was a silent crash. Tantive-III stopped responding with no obvious trigger. The only evidence came from InfluxDB telemetry: Telegraf metrics from the VM flatlined at a specific timestamp, while Node-A's host-level metrics continued briefly before they too stopped. The incident pointed to a VFIO passthrough instability the GPU entered a bad state that propagated up through the PCIe bus and took the entire host with it. Diagnosis was forensic, after the fact, pieced together from time-series gaps.

**VFIO Lockup Incident v2** had a clear culprit: ComfyUI. Configured with Docker's `restart: "unless-stopped"` policy, ComfyUI crashed during an image generation job. Docker faithfully restarted it. ComfyUI crashed again. Docker restarted it again. Each restart attempt allocated VRAM without the previous allocation being properly released. The crash-loop rapidly exhausted all 20GB of VRAM, the GPU driver stalled on PCIe, and Node-A locked up identically to v1.

The fix was surgical: ComfyUI now runs with `restart: "no"`. It must be launched manually. If it crashes, it stays down until a human verifies VRAM is clear and deliberately restarts it. Inconvenient, but a manual launch beats a host-level lockup every time.

**The lesson: same symptoms ≠ same root cause. Always validate after a rebuild.** Incident v1's forensic diagnosis informed the investigation of v2, but the actual mechanism was entirely different. Assuming v2 was "the same VFIO bug" would have missed the crash-loop VRAM exhaustion entirely.

## VRAM and RAM Constraints

VRAM budgeting on the Ada is a zero-sum game. 20GB total, shared across every GPU workload on Tantive-III.

Ollama keeps approximately 12GB warm when a model is loaded the inference server holds model weights in VRAM for fast response times. ComfyUI needs roughly 8GB for Stable Diffusion workloads. That math only works if they don't run simultaneously. Running both at once risks exceeding the 20GB ceiling, triggering OOM conditions, and potentially repeating the crash-loop scenario from v2.

This is why GPU mode switching exists. The fleet treats Ollama inference and ComfyUI image generation as mutually exclusive GPU modes. Scripts handle the transition: verify VRAM is clear, shut down one workload, confirm release, then start the other. The VRAM watchdog monitors allocation in real-time with tiered responses warning at 80% utilization, alerting at 90%, and taking protective action before a hard ceiling hit.

The VM itself has a separate constraint: Tantive-III's RAM is capped at 12GB. Allocating more than 12GB causes a PCIe initialization stall during VM boot the GPU passthrough negotiation fails when the VM's memory footprint exceeds a threshold that interferes with IOMMU mapping. The VM either hangs on boot or starts without GPU access. 12GB was found empirically as the stable ceiling. It stays there.

## Operational Overhead

Running local inference is not a "deploy and forget" workload. The operational burden is real.

GPU mode switching requires coordination. Transitioning between Ollama and ComfyUI means verifying VRAM state, stopping one service, confirming release, and starting the other. The mode-switching scripts and BD-1 slash commands (`/gpu status`, `/gpu mode`) automate the mechanical steps, but a human still decides when to switch.

VRAM monitoring is continuous. The watchdog feeds into n8n orchestration and Discord alerts without it, a VRAM leak could cascade into another lockup. The NMI watchdog at the kernel level provides a last-resort safety net: if the kernel locks up, NMI triggers automatic recovery rather than requiring a physical power cycle.

Manual tuning never ends. New models need VRAM profiling. Quantization choices (Q4_K_M vs Q5_K_M vs Q8) directly impact quality and VRAM footprint. Every model swap is a mini-capacity planning exercise.

## When to Reconsider

The local-first approach holds as long as workloads fit within the Ada's envelope. If BD-1 or fleet workflows ever demand frontier-model quality requiring 70B+ parameters, or if inference latency becomes a bottleneck for time-critical automations, the calculus shifts. A hybrid approach local for private fleet operations, cloud for heavy-lift tasks is the logical pivot. The infrastructure already supports cloud API calls through n8n. The switch isn't architectural; it's a policy decision about which prompts are worth paying for.

Until then, the Ada earns its keep. The trade-offs are known, the failure modes are documented, and the operational overhead is the price of keeping fleet data off someone else's servers.
