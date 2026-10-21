# Implementation: Ghost Squadron

Built 16 to 18 October 2026. State as of 19 October 2026.

## 1. Hardware and OS

- 3x Raspberry Pi 4B, 2GB RAM (1.8Gi usable), 128GB SanDisk High Endurance microSD each
- Ubuntu Server 26.04.1 LTS, kernel 7.0.0-1020-raspi, flashed with Raspberry Pi Imager
- Full `apt` upgrade and reboot on all three before touching K3s
- K3s prerequisites checked: memory cgroup enabled and `vxlan` module loads, so no `cmdline.txt` edit was needed
- Swap left off on purpose
- Tailscale on all three for remote access, key expiry disabled

## 2. Network

**VLAN 60 "Endor"** was created on the UDM as 192.168.60.0/24 with DNS pointing at AdGuard (192.168.1.4) and 1.1.1.1 as fallback. The US-8-150W ports have no spare capacity, so two of the Pis sit behind a NetGear GS108Tv3 managed switch, uplinked to the US-8 on a trunk port.

GS108Tv3 port plan (saved to the switch configuration):

| Port | Device | VLAN 60 | PVID |
|------|--------|---------|------|
| g1, g2 | Pi | untagged | 60 |
| g3 | reserved for a Pi | untagged when used | 1 until moved |
| g4 | Hue bridge | none (VLAN 30 untagged) | 30 |
| g5 to g7 | other devices | none | 1 |
| g8 | uplink to UniFi | tagged | 1 |

The switch was also moved off its factory address to a static management IP, and VLANs 20, 30, 40, 50, and 60 were created on it.

**Static addressing** is set per node in a netplan file (`60-ghost-squadron.yaml`) with a static address, a default route via 192.168.60.1, and DNS pointing at AdGuard with 1.1.1.1 as fallback. cloud-init's network configuration is disabled so netplan is not regenerated. Fixed-IP entries for .11, .12, and .13 also exist in UniFi.

## 3. Firewall (UDM LAN In)

| Rule | Action | Scope |
|------|--------|-------|
| Endor to AdGuard DNS | Accept | TCP/UDP 53 to 192.168.1.4 |
| Endor Block Internal | Drop | Destination list `Fleet-Internal` (the six internal fleet networks) |

Existing rules already cover management access from my workstation and return traffic (established/related). Tested from all three Pis: DNS resolves, the internet works, and the Proxmox web UI on the management network is blocked.

## 4. K3s

| Setting | Value |
|---------|-------|
| Version | v1.36.5+k3s1 |
| Topology | 3 servers, embedded etcd, no dedicated agents |
| First node | Phantom, installed with `--cluster-init` |
| Disabled components | Traefik, servicelb, metrics-server |
| Explicit `--node-ip` | Set on every install command, different on each node |

Result: `kubectl get nodes` shows three Ready nodes with control-plane and etcd roles. Idle, Phantom used about 856Mi with about 976Mi available, running only CoreDNS and the local-path provisioner. That leaves roughly 900Mi per node for workloads.

## 5. Issues Hit and Fixed

| Issue | Cause | Fix |
|-------|-------|-----|
| SSH to the wrong address failed with password errors | The address belonged to the US-8 switch, not a Pi | Found the real IPs in the UniFi client list |
| DHCP lease but no ping to the gateway | Port native network was set to None | Native network set to Default |
| Pi did not pick up a new lease after a port change | A port change does not force a DHCP renewal | Unplugged the cable for five seconds |
| One Pi unreachable behind the GS108Tv3 | Uplink port's tagged VLAN setting was Block All | Changed to Allow All |
| Internet blocked from Endor | First version of the drop rule left Destination as Any | Set destination to the `Fleet-Internal` list |
| K3s join failed with etcd "Peer URLs already exists" | Reused another node's `--node-ip` | Reinstalled with the node's own address |
| K3s join failed with `invalid node-ip` | The shell variable holding the address was empty | Typed the address in directly |
| kubectl "certificate not yet valid" on one node | Pi booted with a stale July clock (no battery-backed RTC) | Set the date by hand and let chrony step it. Did not touch the K3s certificates |

## 6. Not Yet Done

- Rotate the K3s join token (it appeared in plaintext in chat twice) and store the new one in Infisical
- Reboot test, one Pi at a time, checking `date` first and confirming the other nodes stay Ready (etcd needs two of three)
- Time-sync ordering drop-in for the `k3s` service if the reboot test shows a stale clock
- SSH key-only login on all three nodes
- kubectl access from my workstation
- etcd snapshots, with copies stored off the SD cards
- Workloads and fleet integration: see the [README](README.md) status table
