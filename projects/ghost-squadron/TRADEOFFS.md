# Tradeoffs: Ghost Squadron

## Raspberry Pi 4B (2GB) instead of small x86 hosts

**Gain:** Low cost, low power, tiny footprint, and a physically and architecturally separate fault domain (ARM64 hardware, not another Proxmox guest or node).
**Cost:** Memory is the hard limit. Idle, a node has roughly 900Mi free after K3s, so every workload needs explicit requests and limits. Grafana plus VictoriaMetrics probably will not fit next to the other workloads on a shared node.
**Mitigation:** Start with the smallest, highest-value services (secondary AdGuard, Uptime Kuma, alert relay), baseline memory with `kubectl top nodes`, and only then decide whether the metrics replica gets its own node.

## Three servers with embedded etcd instead of one server on SQLite

**Gain:** A high-availability control plane. With three etcd members the cluster survives the loss of any one node, which matches the purpose of the project.
**Cost:** etcd is write-heavy and the nodes boot from microSD. Quorum also means a second Pi must never be rebooted until the first is Ready again.
**Mitigation:** High Endurance cards, one-at-a-time reboot discipline, and etcd snapshots with copies stored off the cards (planned).

## Traefik, servicelb, and metrics-server disabled

**Gain:** Less memory used and no default components I did not choose.
**Cost:** Nothing exposes a service on a LAN address by default, and there is no `kubectl top` until metrics are added back. Exposing AdGuard on port 53 needs another approach.
**Plan:** MetalLB with a small virtual IP range on VLAN 60 for the DNS service.

## Dedicated VLAN with default-deny

**Gain:** If a workload on the cluster is compromised or misbehaves, it cannot reach Proxmox management or the service networks. Only DNS to AdGuard is allowed in, plus internet access.
**Cost:** Every integration needs an explicit rule: Wazuh agents (1514 and 1515), Beszel (45876), cross-VLAN DNS rewrites, and monitoring checks.
**Lesson:** A drop rule with the destination left as Any blocks everything. Check the Destination column after saving.

## NetGear GS108Tv3 as the interim Pi switch

**Gain:** The US-8-150W had no free ports, and the GS108Tv3 was already on hand.
**Cost:** No PoE, so the Pis use their own power supplies. The switch also keeps changes only if the configuration is saved, and its management interface is a second place to maintain.
**Plan:** A used UniFi PoE switch with 802.3af splitters to power the Pis over Ethernet.

## Static addressing in netplan instead of DHCP reservations alone

**Gain:** The cluster never depends on DHCP for its own addresses, and etcd peer addresses stay stable.
**Cost:** A typo in a netplan file breaks a node (one was caught and corrected before apply). `netplan try` rolls back after 120 seconds unless confirmed.

## No battery-backed clock

A Pi has no RTC, so a node can boot with a stale date and K3s will reject its own certificates as "not yet valid". The rule going forward: check `date` before suspecting K3s, fix the clock, and never delete or regenerate certificates to solve a time problem.

## Not covered

A secondary reverse proxy is out of scope (see [PROBLEM](PROBLEM.md)), so Node-C remains a single point of failure for external ingress.
