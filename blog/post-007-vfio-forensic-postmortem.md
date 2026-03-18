# Post 007 - VFIO Forensic Postmortem - Diagnosing a Silent Crash with No Logs

**Tags:** `writeups` `incident-response` `vfio` `gpu-passthrough` `influxdb` `telegraf` `linux-kernel` `pcie` `forensics` `proxmox`

---

Node-A went down on a Sunday morning and left me nothing.

No kernel panic. No crash dump. No syslog entries from the window that mattered. The machine was just off. And when I brought it back up, the logs picked up exactly where the last sync had left them - because I was running **log2ram**, which buffers writes to disk periodically and flushes on clean shutdown. A hard lockup is not a clean shutdown.

This is the story of how I diagnosed a silent Proxmox host crash using only external telemetry, and what I found.

---

## The Environment

Node-A - the Millennium Falcon - is my AI/ML compute node. It runs an NVIDIA RTX 4000 Ada Generation GPU passed through to a VM via VFIO/IOMMU for local LLM inference and RAG workloads.

```
Host:       Node-A (FCM2250 / Millennium Falcon)
CPU:        Intel Core Ultra 9 285K
RAM:        64GB DDR5
GPU:        NVIDIA RTX 4000 Ada (VFIO passthrough to Tantive-III VM)
Hypervisor: Proxmox VE
```

The broader observability stack across the cluster:

- **Telegraf agents** on every host, collecting system metrics every 10 seconds
- **InfluxDB 2.x** on Node-B (`192.168.20.41`), org: `TheAlliance`, bucket: `telegraf`
- **Grafana** on Node-B (`192.168.20.40`), dashboards pulling from InfluxDB

When Node-A locked up, this stack kept running on the other nodes. InfluxDB kept receiving data from everything *except* FCM2250 - and that gap in the time-series data was the first clue.

---

## What I Had (and Didn't Have)

**Available:**
- InfluxDB metrics from FCM2250 up to the moment of lockup
- Metrics from the other nodes (confirming the stack was healthy)
- Post-reboot system uptime - usable as a reference point

**Not available:**
- `/var/log/syslog` from the crash window - buffered in RAM, never written to disk
- `/var/log/kern.log` - same problem
- Kernel panic output - there wasn't one
- MCE (Machine Check Exception) logs - nothing
- pstore / crash dump - empty

log2ram is a useful optimization in normal operation. It reduces disk writes on NVMe by buffering logs in RAM and flushing periodically. In a hard lockup scenario, it erases your evidence - the flush never happens, and nine days of logs vanish with the power state.

---

## The Investigation

### Step 1: Establish the Timeline

The first question: when exactly did Node-A stop responding?

```flux
from(bucket: "telegraf")
  |> range(start: 2026-02-09T15:00:00Z, stop: 2026-02-09T16:00:00Z)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> filter(fn: (r) => r.host == "FCM2250")
  |> filter(fn: (r) => r.cpu == "cpu-total")
  |> filter(fn: (r) => r._field == "usage_idle")
  |> last()
```

**Result:** The last data point from FCM2250 was at **15:54:50 UTC on February 9, 2026**. After that timestamp, the time-series data has a clean gap - no data, no partial writes, no corrupted entries. The Telegraf agent simply stopped sending.

### Step 2: Characterize the System State at Crash

What was the system doing at the moment it locked up?

```flux
from(bucket: "telegraf")
  |> range(start: 2026-02-09T15:50:00Z, stop: 2026-02-09T15:55:00Z)
  |> filter(fn: (r) => r.host == "FCM2250")
  |> filter(fn: (r) => r._measurement == "cpu" or r._measurement == "mem" or r._measurement == "diskio")
  |> aggregateWindow(every: 10s, fn: mean)
```

**Findings:**

| Metric | Value at 15:54:50 | Normal Range |
|--------|:-----------------:|:------------:|
| CPU idle | ~95% | 85-98% |
| Memory used | ~18GB / 64GB | Normal |
| Disk I/O | Near zero | Normal for idle |
| Network | Minimal | Normal |

The system was **idle**. No CPU spike, no memory pressure, no I/O storm. Whatever caused the lockup wasn't triggered by load.

### Step 3: Verify Other Nodes Were Healthy

Confirm the crash was isolated to Node-A, not a cluster-wide event:

```flux
from(bucket: "telegraf")
  |> range(start: 2026-02-09T15:50:00Z, stop: 2026-02-09T16:00:00Z)
  |> filter(fn: (r) => r.host != "FCM2250")
  |> filter(fn: (r) => r._measurement == "cpu")
  |> filter(fn: (r) => r.cpu == "cpu-total")
```

**Result:** Node-B and Node-C continued reporting normally through the entire window. No gaps, no anomalies. The Proxmox Corosync cluster detected Node-A's absence but the other nodes were unaffected.

### Step 4: Check for GPU Metrics Anomalies

Node-A runs the RTX 4000 Ada via VFIO passthrough. Telegraf collects `nvidia_smi` metrics from inside the Tantive-III VM:

```flux
from(bucket: "telegraf")
  |> range(start: 2026-02-09T15:00:00Z, stop: 2026-02-09T15:55:00Z)
  |> filter(fn: (r) => r._measurement == "nvidia_smi")
  |> last()
```

The GPU metrics stopped at the same timestamp as the host metrics. No temperature spike, no utilization anomaly, no error state reported before the cutoff. The GPU wasn't overheating or under load - it was idle.

### Step 5: Post-Reboot Analysis

After power cycling Node-A:

```bash
# Check for kernel crash artifacts
ls -la /sys/fs/pstore/
# Empty - no crash dump

# Check for MCE errors
mcelog --client
# No errors

# Check journal (only shows post-boot entries)
journalctl --since "2026-02-09" | head -50
# First entry is the boot sequence - nothing from before the crash

# Check dmesg for hardware errors
dmesg | grep -i -E "error|fault|fail|mce|nmi"
# Clean - no hardware errors on this boot
```

No local evidence survived. The investigation is entirely dependent on the external InfluxDB telemetry.

---

## Root Cause

**PCIe bus stall induced by the NVIDIA GPU under VFIO passthrough.**

The evidence chain:

1. System was idle at crash time - rules out load-induced failure
2. No MCE errors - rules out memory hardware fault
3. No kernel panic - rules out software crash (a panic would have been logged to pstore)
4. No temperature anomaly - rules out thermal shutdown
5. GPU was in VFIO passthrough mode - the only hardware component with a complex, stateful interface to the PCIe bus
6. The lockup was instantaneous (no degradation in the 10-second metric windows) - consistent with a bus-level stall, not a gradual failure

NVIDIA GPUs in VFIO passthrough can trigger PCIe bus stalls during power state transitions (D0 ↔ D3). When the GPU attempts to change power states and the transition fails, the PCIe bus hangs. Because the bus is shared with other critical devices, the entire system locks up. No interrupt fires, no NMI triggers, no kernel code path has a chance to log anything.

This is a known class of issue with VFIO GPU passthrough, particularly with NVIDIA cards that aggressively manage power states.

---

## Mitigation

Added kernel boot parameters to prevent the GPU from entering the power states that trigger the stall:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_aspm=off pci=noaer"
```

- `pcie_aspm=off` - Disables PCIe Active State Power Management. Prevents the GPU from entering low-power states that can trigger the bus stall.
- `pci=noaer` - Disables Advanced Error Reporting. Prevents kernel log spam from GPU power state transitions that aren't actual errors.

```bash
update-grub
reboot
```

### Additional Mitigation: Disable log2ram

Removed log2ram from Node-A. The disk write savings aren't worth the forensic blindness during hardware failures. Persistent logging means the next crash - if it happens - will leave local evidence.

---

## Stability Result

Node-A has been stable since applying the kernel parameters. No further lockups as of the time of writing.

---

## Lessons Learned

1. **External telemetry is not optional.** If InfluxDB hadn't been running on a separate node, this crash would have been completely undiagnosable. The TIG stack ([Post 017](/post017/)) exists for exactly this scenario.

2. **log2ram is a liability on critical nodes.** The performance benefit of buffering logs in RAM is negligible on NVMe. The forensic cost of losing 9 days of logs during a hard lockup is enormous.

3. **GPU passthrough adds hardware-level failure modes.** VFIO is not a transparent abstraction - it exposes the host to PCIe bus behavior from the guest's GPU driver. Power management issues in the guest can crash the host.

4. **An idle system can still crash.** The instinct is to look for what triggered the failure - a load spike, a rogue process, a memory leak. Sometimes the trigger is a hardware state transition that happens autonomously, with no external stimulus.

---

## Investigation Timeline

| Time | Event |
|------|-------|
| 15:54:50 UTC | Last Telegraf data point from FCM2250 |
| 15:54:50+ | Node-A hard locks - no local artifacts |
| ~16:30 UTC | Noticed Node-A unreachable |
| ~16:35 UTC | Power cycled Node-A |
| ~16:40 UTC | Node-A reboots - pstore empty, journal starts fresh |
| ~17:00 UTC | Began InfluxDB forensic queries |
| ~17:30 UTC | Established timeline and ruled out software causes |
| ~18:00 UTC | Identified VFIO/PCIe as probable root cause |
| ~18:15 UTC | Applied kernel parameters, rebooted |

---

*Related: [Post 017 - TIG Stack Observability](/post017/) (the telemetry that saved this investigation) | [Post 018 - GPU Passthrough Setup](/post018/) (VFIO configuration and kernel parameters)*
