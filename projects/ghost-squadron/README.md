# Ghost Squadron: K3s Resilience Layer

A three-node K3s cluster on Raspberry Pi 4B hardware, isolated on its own VLAN, built to remove single points of failure from the Alliance Fleet's Proxmox side.

**Stack:** K3s v1.36.5 (3 servers, embedded etcd) | Ubuntu Server 26.04 LTS | Raspberry Pi 4B (2GB) | UniFi VLAN 60 "Endor" | NetGear GS108Tv3 | Tailscale

---

## Why It Exists

The fleet's most important shared services sit on a few Proxmox nodes. DNS and the reverse proxy live on Node-C, and the alert pipeline and uptime monitoring live on Node-B. When one of those nodes goes down, the fleet loses DNS or loses the ability to tell me it is down. Ghost Squadron runs secondary copies of those services on separate hardware, a separate network segment, and a different orchestrator.

## Status (19 Oct 2026)

| Area | State |
|------|-------|
| Hardware, OS, static addressing | Done |
| VLAN 60, UDM firewall rules, GS108Tv3 port plan | Done and tested |
| Tailscale, time sync | Done |
| K3s cluster (3 servers, embedded etcd) | Up, all nodes Ready |
| Join token rotation, SSH key-only login, reboot test, etcd snapshot copies off the SD cards | In progress |
| First workloads (secondary AdGuard, Uptime Kuma, alert relay, Grafana + VictoriaMetrics) | Next |
| Fleet integration (Beszel, Wazuh agents, NetBox, Homepage) | Next |

The cluster is up and healthy. It is not yet running the workloads that justify it, and this repo says so on purpose.

## The Nodes

| Node | Role | Address (VLAN 60) | RAM |
|------|------|-------------------|-----|
| Phantom | K3s server, `--cluster-init` | 192.168.60.11 | 2GB |
| Gauntlet | K3s server | 192.168.60.12 | 2GB |
| Nightbrother | K3s server | 192.168.60.13 | 2GB |

All three nodes run the control plane, etcd, and workloads. With three etcd members the cluster tolerates the loss of one node.

## Documentation

1. **[PROBLEM](PROBLEM.md)**: the single points of failure this layer targets.
2. **[IMPLEMENTATION](IMPLEMENTATION.md)**: hardware, network, firewall, K3s install, and the issues hit along the way.
3. **[TRADEOFFS](TRADEOFFS.md)**: why Raspberry Pis, why embedded etcd, and what 2GB of RAM costs.

## Skills Demonstrated

`Kubernetes (K3s)` `etcd HA` `VLAN design` `Firewall policy (default-deny)` `Linux administration (Ubuntu, netplan, chrony)` `Resilience and fault-domain design` `Troubleshooting and documentation`
