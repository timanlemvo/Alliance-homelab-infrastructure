# Proxmox Homelab: Cloud, Security & Platform Engineering Lab

Production-grade infrastructure running 30+ services across a 4-node Proxmox cluster and a 3-node K3s (Kubernetes) cluster on Raspberry Pi, with zero-trust identity, security monitoring, full-stack observability, GPU-accelerated AI workloads, and hybrid Active Directory with Entra Connect sync.

---

## Overview

This is a working simulation of enterprise infrastructure, built by applying lessons from 8 years managing 200+ users at Team Liquid, Stagwell, and CAA. Every design decision (VLAN segmentation, centralized identity, observability pipelines, threat detection, hybrid identity) mirrors production standards implemented professionally.

`Linux Administration` `Proxmox VE Clustering` `Kubernetes (K3s)` `Active Directory` `Hybrid Identity (Entra Connect)` `VLAN Segmentation` `Firewall Policy` `SIEM/XDR (Wazuh)` `SSO/IAM (Authentik)` `Observability (Telegraf → InfluxDB → Grafana)` `Zero-Trust (Tailscale)` `GPU Passthrough (VFIO/IOMMU)` `Reverse Proxy & TLS` `Threat Detection & Response` `Incident Forensics` `Infrastructure Documentation`

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│      PROXMOX VE CLUSTER (4 Nodes) │ Corosync 4-vote quorum │ PVE 9.2.21      │
├──────────────────────────────────────┬───────────────────────────────────────┤
│ NODE-A  Millennium Falcon (FCM2250)  │ NODE-B  CR90 Corvette (QCM1255)       │
│ Core Ultra 9 285 | 64GB DDR5         │ Ryzen 7 PRO 8700GE | 64GB DDR5        │
│ 2x 1TB NVMe | RTX 4000 SFF Ada 20GB  │ 2x 1TB NVMe | USB backup disks        │
│ ROLE: AI/ML compute, DFIR lab        │ ROLE: Identity, data, secrets         │
│                                      │                                       │
│ Tantive-III: Ollama, OpenWebUI,      │ Home One: Authentik, Portainer,       │
│   ComfyUI, AnythingLLM, Beszel hub   │   Garage, Samba                       │
│ Canto-Bight (Windows 11)             │ Phoenix-Nest: n8n, Vaultwarden,       │
│ MaxRebo: AzuraCast radio             │   Homepage, Umami                     │
│ DFIR VMs: Raddus, Scimitar           │ InfluxDB, Grafana, Uptime Kuma,       │
│                                      │   Infisical, RustDesk, K-2SO          │
├──────────────────────────────────────┼───────────────────────────────────────┤
│ NODE-C  Gozanti Cruiser (7050)       │ NODE-D  GR-75 Transport (7040)        │
│ i7-7700T | 32GB DDR4                 │ i5-6500T | 24GB DDR4                  │
│ 1TB NVMe + 1TB SATA | bonded NICs    │ 1TB NVMe + 2TB SATA                   │
│ ROLE: Network edge, security         │ ROLE: AD, agents, docs, overflow      │
│                                      │                                       │
│ AdGuard Home (DNS)                   │ alliance-dc01 (AD DS)                 │
│ Nginx Proxy Manager                  │ Coruscant: Jocasta agent              │
│ Wazuh SIEM                           │ Yavin-IV: BookStack, Gitea,           │
│ Vikunja                              │   NetBox                              │
│ Fulcrum (Bot-Net node)               │ Home Assistant                        │
└──────────────────────────────────────┴───────────────────────────────────────┘
```
```
┌──────────────────────────────────────────────────────────────────────────────┐
│            GHOST SQUADRON: K3s CLUSTER (3 servers, embedded etcd)            │
│          VLAN 60 Endor │ 192.168.60.0/24 │ Ubuntu Server 26.04 LTS           │
├─────────────────────────┬─────────────────────────┬──────────────────────────┤
│ PHANTOM (cluster-init)  │ GAUNTLET                │ NIGHTBROTHER             │
│ Pi 4B │ 2GB │ .11       │ Pi 4B │ 2GB │ .12       │ Pi 4B │ 2GB │ .13        │
├─────────────────────────┴─────────────────────────┴──────────────────────────┤
│ K3s v1.36.5+k3s1 │ Traefik, servicelb, metrics-server disabled               │
│ ROLE: Resilience layer (secondary DNS, monitoring, alert relay)              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**[Live Architecture Diagram →](https://timanlemvo.github.io/Alliance-homelab-infrastructure/)**

[![Alliance Fleet Architecture](https://timanlemvo.github.io/Alliance-homelab-infrastructure/alliance-fleet-architecture.html)](https://timanlemvo.github.io/Alliance-homelab-infrastructure/)

### Hardware Specifications

| Node | Codename | Processor | Memory | Storage | Special Hardware |
|------|----------|-----------|--------|---------|------------------|
| Node-A | Millennium Falcon | Intel Core Ultra 9 285 | 64GB DDR5-6400 | 2x 1TB NVMe | RTX 4000 SFF Ada 20GB (VFIO passthrough) |
| Node-B | CR90 Corvette | AMD Ryzen 7 PRO 8700GE | 64GB DDR5-4800 | 2x 1TB NVMe + USB backup disks | Local backup target (ext-backup NFS), identity and data services |
| Node-C | Gozanti Cruiser | Intel Core i7-7700T | 32GB DDR4-2133 | 1TB NVMe + 1TB SATA | Dual NIC bonded active-backup (I219-LM primary, RTL8125 standby) |
| Node-D | GR-75 Transport | Intel Core i5-6500T | 24GB DDR4 | 1TB NVMe + 2TB SATA | Micro form factor, low power, fourth quorum vote |
| Ghost Squadron (x3) | Phantom, Gauntlet, Nightbrother | Broadcom BCM2711 (Cortex-A72) | 2GB LPDDR4 each | 128GB SanDisk High Endurance microSD | Raspberry Pi 4B, K3s servers on VLAN 60 |

Each node has a defined role. Node-A carries the GPU for AI/ML inference (20GB VRAM) and the DFIR lab, and is otherwise lightly loaded. Node-B holds the identity, secrets, data, and observability services (Authentik, Infisical, InfluxDB, Grafana) and the local backup target. Node-C is the network edge: DNS, the reverse proxy, and the SIEM all live there, which makes it the fleet's availability keystone and the reason Ghost Squadron exists. Node-D (added October 2026) is a utility node. It took the domain controller, the Jocasta agent, the documentation stack, and Home Assistant off Node-B, and its fourth quorum vote lets the cluster lose any single node and keep quorum.

Ghost Squadron is a separate three-node K3s cluster on Raspberry Pi 4B hardware. It runs on its own VLAN, its own orchestrator, and its own hardware, so it can host secondary copies of fleet-critical services (DNS, uptime monitoring, alert relay) that stay up when a Proxmox node does not. See [Ghost Squadron](projects/ghost-squadron/).

---

## Network Architecture

### VLAN Structure

```
╔══════════════════════════════════════════════════════════════════════════╗
║                              VLAN TOPOLOGY                               ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  VLAN 1 - MANAGEMENT / TATOOINE (192.168.1.0/24)                         ║
║  ├─ .1    UniFi Dream Machine (gateway, firewall)                        ║
║  ├─ .2    UniFi US-8-150W (switch)                                       ║
║  ├─ .4    AdGuard Home (LXC 100, Node-C)                                 ║
║  ├─ .10-.13  Proxmox Node-A, B, C, D                                     ║
║  ├─ .101  Nginx Proxy Manager (LXC 101, Node-C)                          ║
║  └─ Nodes sit in UniFi's Default zone, static assignments                ║
║                                                                          ║
║  VLAN 20 - SERVICES / NABOO (192.168.20.0/24)                            ║
║  ├─ .10  Home One (Authentik, Portainer, Garage)                         ║
║  ├─ .20  Tantive-III (Ollama, OpenWebUI, ComfyUI)                        ║
║  ├─ .30  Wazuh SIEM       .40 Grafana     .41 InfluxDB                   ║
║  ├─ .50  MaxRebo (AzuraCast radio)                                       ║
║  ├─ .60  Yavin-IV (BookStack, Gitea, NetBox)                             ║
║  ├─ .61  Uptime Kuma      .81 RustDesk                                   ║
║  ├─ .70  Phoenix-Nest (n8n, Vaultwarden, Homepage, Umami)                ║
║  ├─ .80  Stinger Mantis (K-2SO)   .82 Coruscant (Jocasta)                ║
║  ├─ .86  Canto-Bight   .115 Vikunja   .117 Infisical                     ║
║  └─ ALLIANCE-DC01 (VM 300, Node-D)                                       ║
║                                                                          ║
║  VLAN 30 - IoT / MUSTAFAR (192.168.30.0/24)                              ║
║  ├─ .10  Home Assistant (VM 116, Node-D)                                 ║
║  ├─ Tapo switches, LG TV, thermostat                                     ║
║  └─ No internet except Home Assistant; drops to Mgmt, Services, DMZ      ║
║                                                                          ║
║  VLAN 40 - DMZ / SCARIF (192.168.40.0/24)                                ║
║  ├─ Reserved for external ingress                                        ║
║  ├─ First tenant planned: cloudflared (today on Phoenix-Nest)            ║
║  └─ No DHCP, static assignments only                                     ║
║                                                                          ║
║  VLAN 50 - BOT-NET / SECURITY SANDBOX (192.168.50.0/28)                  ║
║  ├─ .10  Fulcrum (VM 204, console only, no sshd)                         ║
║  ├─ Scoped allow-list: Proxmox API, Ollama, InfluxDB, Wazuh API,         ║
║  │  Uptime Kuma, DNS; reject to IoT and DMZ                              ║
║  └─ No DHCP, static assignments only                                     ║
║                                                                          ║
║  VLAN 60 - ENDOR / GHOST SQUADRON (192.168.60.0/24)                      ║
║  ├─ .11-.13  Phantom, Gauntlet, Nightbrother (K3s servers)               ║
║  ├─ Static IPs via netplan, fixed-IP entries in UniFi                    ║
║  └─ Default-deny to fleet networks, DNS to AdGuard allowed               ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

| VLAN | Codename | Name | Subnet | Gateway | DHCP | Purpose |
|------|----------|------|--------|---------|------|---------|
| 1 | Tatooine | Management | 192.168.1.0/24 | 192.168.1.1 | Disabled | Hypervisor & infrastructure management |
| 20 | Naboo | Services | 192.168.20.0/24 | 192.168.20.1 | .100-.200 | Application workloads |
| 30 | Mustafar | IoT | 192.168.30.0/24 | 192.168.30.1 | .100-.200 | Smart home, fully isolated |
| 40 | Scarif | DMZ | 192.168.40.0/24 | 192.168.40.1 | Disabled | Reserved for external ingress (cloudflared planned) |
| 50 | Bot-Net | Security Sandbox | 192.168.50.0/28 | 192.168.50.1 | Disabled | Isolated security sandbox (Fulcrum VM204) |
| 60 | Endor | Ghost Squadron | 192.168.60.0/24 | 192.168.60.1 | Enabled (Pis use static IPs) | K3s cluster on Raspberry Pi 4B |

DHCP is disabled on Management, DMZ, and Bot-Net. Management contains hypervisor interfaces. Unauthorized access means full infrastructure compromise. DMZ is the external attack surface. Bot-Net is an isolated sandbox with a scoped allow-list (Proxmox API, Ollama, InfluxDB, Wazuh API, Uptime Kuma, DNS) and a reject rule toward IoT and DMZ. Every host on these VLANs must be explicitly provisioned.

Endor (VLAN 60) hosts the Ghost Squadron K3s cluster. It is default-deny toward the fleet's internal networks: the only inbound hole is DNS to AdGuard, and a drop rule blocks the six internal networks. Any new integration (Wazuh agents, Beszel, monitoring checks) needs an explicit rule.

IoT is fully isolated. Consumer IoT devices are notoriously insecure. VLAN 30 has no internet access (only Home Assistant is allowed out) and cannot initiate connections to Management, Services, or DMZ.

### Network Hardware

| Device | Model | Role |
|--------|-------|------|
| Gateway/Firewall | UniFi Dream Machine | Routing, inter-VLAN firewall, VLAN management, threat management |
| Switch | UniFi US-8-150W | PoE managed switch, VLAN trunking to all nodes |
| WiFi AP | UniFi Beacon HD | Wireless connectivity |
| Switch (Ghost Squadron) | NetGear GS108Tv3 | Managed 8-port switch carrying VLAN 60 to the Raspberry Pis, trunked to the US-8-150W |

Previously ran OPNsense as a dedicated firewall, but persistent issues with the ISP modem releasing IP addresses during the dual-gateway setup led to reliability problems. Consolidated to the UniFi Dream Machine as the single gateway/firewall. The tradeoff is less granular firewall API access. Automated threat response is being rebuilt for the UDM platform.

### Traffic Flow: External Request

```
Internet
  │
  ▼
Cloudflare edge (tunnel "alliance-fleet", *.tima.dev)
  │
  ▼
cloudflared (Phoenix-Nest, VLAN 20; planned move to VLAN 40 / DMZ)
  │
  ▼
Nginx Proxy Manager (192.168.1.101, TLS termination)
  │
  ▼
Backend service on VLAN 20 (Authentik SSO on registered applications)
```

Internal clients skip the tunnel: AdGuard rewrites `*.tima.dev` to the NPM host, so names resolve and terminate TLS the same way on the LAN.

### Remote Access: Tailscale

```
┌─────────────────────────────────────────────────────────────┐
│                    TAILSCALE OVERLAY                        │
├─────────────────────────────────────────────────────────────┤
│  100.x.x.10 --> Node-A (Millennium Falcon)                  │
│  100.x.x.11 --> Node-B (CR90 Corvette)                      │
│  100.x.x.12 --> Node-C (Gozanti Cruiser) <- Subnet Router   │
│  100.x.x.13 --> Node-D (GR-75 Transport)                    │
│  100.x.x.20 --> Selected VMs/Containers                     │
└─────────────────────────────────────────────────────────────┘
```

Node-C advertises the management subnet to authorized Tailscale clients. Remote devices enrolled in the tailnet can reach infrastructure without any ports exposed to the public internet.

---

## Security Stack

```
Layer 1  - PERIMETER:    UniFi Dream Machine firewall + threat management
Layer 2  - NETWORK:      VLAN segmentation + inter-VLAN firewall rules
Layer 3  - ACCESS:       Tailscale overlay (admin) + Cloudflare Tunnel (public services)
Layer 4  - IDENTITY:     Authentik SSO + MFA (TOTP/WebAuthn), 7 apps, expanding
Layer 5  - INGRESS:      Nginx Proxy Manager (TLS termination, rate limiting)
Layer 6  - DNS:          AdGuard filtering (ad/tracker/malware domain blocking)
Layer 7  - SECRETS:      Infisical (machine identities) + Vaultwarden
Layer 8  - DETECTION:    Wazuh SIEM/XDR (log aggregation, threat detection, FIM)
Layer 9  - ALERTING:     n8n automation (Wazuh alerts → Discord notifications)
Layer 10 - MONITORING:   Telegraf → InfluxDB → Grafana + UptimeKuma
```

---

## Implemented Capabilities

### Hybrid Active Directory & Identity

**Stack:** Windows Server 2022 (AD DS) → Entra Connect → Azure AD (Password Hash Sync)

Domain controller (`ALLIANCE-DC01`) built from scratch on Node-A, then migrated to Node-B and, in October 2026, to Node-D as the fleet was rebalanced. Forest `alliance.lab` with full OU structure, cascading GPO inheritance, and break/fix scenario testing. Entra Connect configured with Password Hash Sync syncing identities to an existing Azure tenant, closing the on-prem to cloud identity loop. Domain-joined Windows 11 Pro workstation (Canto-Bight) used for end-to-end GPO validation. Includes a full troubleshooting arc: Server Core vs Desktop Experience root cause diagnosis, VM rebuild decision, cross-VLAN domain join DNS fix, and Proxmox migration CD/DVD blocker resolution.

[Active Directory Hybrid Identity Lab →](projects/active-directory-lab/)

### Security Monitoring & Threat Detection

**Stack:** Wazuh SIEM/XDR → n8n → Discord alerting

Wazuh deployed with log aggregation, file integrity monitoring, and rule-based threat detection across the cluster. n8n processes Wazuh webhooks and pushes alerts to Discord for real-time visibility. Previously included automated IP blocking via OPNsense API. Automated enforcement is being rebuilt for the UniFi platform. Detection and alerting remain fully operational.

[SIEM Automation Pipeline →](projects/security-monitoring/)

### Zero-Trust Identity & Access Management

**Stack:** Authentik → OIDC/SAML → 7 registered applications (Beszel, BookStack, Gitea, Grafana, NetBox, Outline, Portainer), MFA enforced (TOTP + WebAuthn)

Single SSO login for the registered applications, with MFA enforced at the identity provider. The most privileged consoles (Proxmox, Vaultwarden, Infisical, n8n, Wazuh) are still on local auth, and an SSO expansion sprint is under way, starting with an AD LDAP source from ALLIANCE-DC01. Login audit trail forwarded to Wazuh.

[Zero-Trust Identity Platform →](projects/identity-access/)

### Full-Stack Observability

**Stack:** Telegraf → InfluxDB 2.x (Flux) → Grafana → Discord alerts

10-second metric granularity across all 4 nodes. This pipeline was the sole reason a hard lockup root cause analysis was possible when all local logs were destroyed by `log2ram`.

[VFIO Lockup Forensics →](https://github.com/timanlemvo/technical-writeups/tree/main/proxmox-vfio-lockup-forensics)

### Network Segmentation

**Stack:** UniFi Dream Machine + 6 VLANs

Management, Services, IoT, DMZ, Bot-Net, and the Endor K3s segment fully segmented. IoT devices cannot reach infrastructure. DMZ is the only public-facing surface. DHCP disabled on high-trust and high-exposure VLANs.

### GPU-Accelerated AI/ML Platform

**Stack:** Ollama + ComfyUI + AnythingLLM, RTX 4000 Ada via VFIO passthrough

Local LLM inference with no external API dependency. RAG pipeline for querying internal documents. GPU stable post-PCIe mitigation.

[GPU AI Platform →](projects/ai-platform/)

### Container Orchestration: Ghost Squadron (K3s)

**Stack:** K3s v1.36.5 (3 servers, embedded etcd) on Raspberry Pi 4B, Ubuntu Server 26.04 LTS, VLAN 60

A three-node high-availability K3s cluster built as a resilience layer for the Proxmox fleet. All three nodes run the control plane, etcd, and workloads, and the cluster tolerates the loss of one node. The segment is default-deny to the fleet's internal networks, with firewall behavior tested from every node. Status: the cluster is up with all nodes Ready; hardening (token rotation, key-only SSH, reboot test, etcd snapshots) and the first workloads (secondary AdGuard, Uptime Kuma, Discord alert relay) are next.

[Ghost Squadron →](projects/ghost-squadron/)

### Automation & Orchestration

**Stack:** n8n workflows + Docker Compose

Event-driven alerting, infrastructure notifications to Discord, HomeAssistant integration on isolated IoT VLAN.

---

## Projects

| Project | Problem Solved | Tech | Evidence |
|---------|----------------|------|----------|
| [Active Directory Hybrid Identity Lab](projects/active-directory-lab/) | No on-prem AD or hybrid identity in the fleet | Windows Server 2022, AD DS, GPO, Entra Connect, PHS | Domain controller, cascading GPOs, domain-joined Win11 workstation, Azure sync |
| [Zero-Trust Identity](projects/identity-access/) | Password sprawl, no MFA | Authentik OIDC/SAML | 7 applications on SSO with MFA, expansion in progress, audit trail |
| [SIEM Pipeline](projects/security-monitoring/) | Manual threat monitoring doesn't scale | Wazuh + n8n + Discord | Automated detection and alerting, prior auto-blocking |
| [Ghost Squadron (K3s)](projects/ghost-squadron/) | DNS, alerting, and monitoring each depend on a single Proxmox node | K3s, embedded etcd, Raspberry Pi 4B, UniFi VLANs, GS108Tv3 | 3-node HA control plane Ready, isolated VLAN 60, firewall policy tested from every node |
| [GPU AI Platform](projects/ai-platform/) | Local LLM inference | Ollama + RTX 4000 Ada | On-device inference, RAG pipeline in progress |
| [Job Radar](projects/job-radar/) | Manual job searching across multiple platforms | n8n, LinkedIn API, Indeed API, 3+ APIs, Discord embeds | Automated job discovery pipeline with deduplication and real-time Discord delivery |
| [BD-1 Discord Assistant](projects/bd1-job-tracker/) | Fragmented tooling for job search, cert study, and project tracking | BD-1 bot, PostgreSQL, AnythingLLM, n8n, Discord | Feature module expanding BD-1 with project tracking, cert study commands, and job search integration |
| [VFIO Lockup Forensics](https://github.com/timanlemvo/technical-writeups/tree/main/proxmox-vfio-lockup-forensics) | Silent host crash, zero local logs | Telegraf + InfluxDB + Flux | Root cause found via external telemetry alone |

---

## Incident Response

### [Diagnosing a Silent Hard Lockup on a Proxmox VFIO Node](https://github.com/timanlemvo/technical-writeups/tree/main/proxmox-vfio-lockup-forensics)

Node-A (Millennium Falcon) suffered a complete hard lockup with zero local crash artifacts. No kernel panic, no pstore dump, no journal entries. `log2ram` held all logs in RAM and the instantaneous failure prevented a sync to disk. By pivoting to external Telegraf metrics in InfluxDB, a second-by-second timeline was reconstructed using Flux queries. Every software cause was eliminated (CPU idle at 99.8%, memory at 7.4%, zero network errors), and the failure was traced to a PCIe bus stall from the NVIDIA GPU under VFIO passthrough. Kernel mitigations applied (`pcie_aspm=off`, `pci=noaer`), `log2ram` disabled, node stable since.

---

## Enterprise Mapping

| Homelab Capability | Enterprise Equivalent | Role Relevance |
|-------------------|----------------------|----------------|
| AD DS + Entra Connect + GPO | On-prem AD synced to Azure AD | Cloud Engineer, IAM Engineer, Sysadmin |
| Authentik SSO + MFA | Okta / Azure AD / Ping | Identity Engineer, Security |
| Wazuh SIEM + n8n alerting | Splunk / Sentinel / QRadar | Security Engineer, SOC |
| Grafana + Telegraf + InfluxDB | Datadog / New Relic / Prometheus | SRE, Platform Engineering |
| VLAN segmentation + UDM firewall | Cisco / Palo Alto / Fortinet | Network Engineer, Cloud Security |
| Proxmox 4-node cluster | VMware / Hyper-V / Nutanix | Systems Engineer, Virtualization |
| K3s HA cluster (embedded etcd) | AKS / EKS / GKE | Cloud Engineer, DevOps, Platform Engineering |
| Ollama + GPU passthrough | SageMaker / ML Platform | MLOps, AI Infrastructure |
| Tailscale zero-trust | Zscaler / Cloudflare Access | Cloud Security, Zero Trust |
| Incident forensics writeup | Production postmortem | SRE, Incident Response |

---

## Current State & Roadmap

### Implemented

- 4-node Proxmox cluster with corosync quorum
- 6-VLAN segmentation (Management/Tatooine, Services/Naboo, IoT/Mustafar, DMZ/Scarif, Security Sandbox/Bot-Net, Endor/Ghost Squadron)
- Ghost Squadron: 3-node K3s cluster (embedded etcd HA) on Raspberry Pi 4B, isolated on VLAN 60
- Active Directory domain controller (ALLIANCE-DC01) on Node-D (migrated from Node-B, October 2026) with cascading GPOs and break/fix testing
- Entra Connect hybrid identity sync (Password Hash Sync) to Azure tenant
- Centralized SSO via Authentik with MFA on 7 registered applications (expansion to privileged consoles in progress)
- Infisical secrets management (secrets.tima.dev) with machine identities; credentials for K-2SO, Jocasta, Outline, and the Azure backup job migrated off disk
- Wazuh SIEM deployed with detection rules, FIM, and log aggregation
- Full observability pipeline (Telegraf → InfluxDB → Grafana)
- Tailscale zero-trust remote access with subnet routing
- GPU passthrough with PCIe stability mitigations
- DNS filtering via AdGuard
- Reverse proxy with TLS termination via NPM, public access through a Cloudflare Tunnel
- Node-C NIC bond (active-backup) and all four nodes patched to PVE 9.2.21
- Self-hosted services: Garage (S3), Outline, Vikunja, Umami analytics, AzuraCast radio, RustDesk, Homepage
- Self-hosted password management (Vaultwarden)
- Azure Blob offsite backups via azcopy (nightly cron, SAS-scoped)
- Jocasta AI agent on Hermes framework (local Ollama + Claude API elevation)
- Alliance Fleet Codex (BookStack + Gitea + NetBox, nightly n8n sync)
- Beszel server monitoring across all nodes
- GPU thermal safety with Ollama circuit breaker
- n8n automation platform (20+ active workflows)
- Home Assistant on VLAN 30
- RustDesk self-hosted remote desktop
- K-2SO and BD-1 Discord bots (BD-1 archived)

### In Progress

- AZ-104 certification prep
- Ghost Squadron hardening: join token rotation, reboot test, SSH key-only login, etcd snapshots off the SD cards
- Ghost Squadron first workloads: secondary AdGuard, secondary Uptime Kuma, Discord alert relay
- Wazuh agent tuning and custom detection rule expansion
- Inter-VLAN firewall rule hardening
- Tailscale ACL policy refinement
- Second domain controller on Node-C for AD redundancy
- DFIR forensics lab (VMs 900-904 range on Node-A) and its dedicated VLAN
- Authentik SSO expansion (AD LDAP source, privileged consoles)
- Grafana dashboard rebuild (hub-and-spoke)

### Planned

| Initiative | Why |
|-----------|-----|
| Ghost Squadron expansion | Fourth node with a Wazuh agent and UDM syslog role, then a PoE switch to power the Pis over Ethernet |
| Terraform for VM provisioning | Full IaC, GitOps workflow |
| Grafana alerting rules | CPU, memory, disk, service availability thresholds |

### Completed (previously planned)

- Offsite encrypted backups: Azure Blob via azcopy, nightly cron from Node-B
- Node-D (GR-75 Transport) added to cluster, Jocasta agent migrated from Node-B
- n8n automated threat response with UniFi IP blocking via AbuseIPDB enrichment

---

## Repository Structure

```
Alliance-homelab-infrastructure/
├── README.md                              <- You are here
├── projects/
│   ├── active-directory-lab/              <- Hybrid AD + Entra Connect identity lab
│   │   ├── README.md
│   │   ├── docs/
│   │   │   ├── PROBLEM.md
│   │   │   ├── IMPLEMENTATION.md
│   │   │   └── TRADEOFFS.md
│   │   └── blog/
│   │       └── active-directory-hybrid-identity-lab.md
│   ├── security-monitoring/               <- SIEM automation pipeline
│   │   ├── PROBLEM.md
│   │   ├── IMPLEMENTATION.md
│   │   ├── TRADEOFFS.md
│   │   ├── wazuh/
│   │   ├── n8n/
│   │   └── docker-compose.yml
│   ├── identity-access/                   <- Zero-trust identity platform
│   │   ├── PROBLEM.md
│   │   ├── IMPLEMENTATION.md
│   │   ├── TRADEOFFS.md
│   │   ├── authentik/
│   │   └── integrations/
│   ├── ai-platform/                       <- GPU AI/ML platform
│   │   ├── PROBLEM.md
│   │   ├── IMPLEMENTATION.md
│   │   ├── TRADEOFFS.md
│   │   ├── ollama/
│   │   └── rag/
│   ├── ghost-squadron/                    <- K3s resilience cluster on Raspberry Pi 4B
│   │   ├── README.md
│   │   ├── PROBLEM.md
│   │   ├── IMPLEMENTATION.md
│   │   └── TRADEOFFS.md
│   ├── bd1-job-tracker/                   <- BD-1 Discord assistant feature expansion
│   │   ├── PROBLEM.md
│   │   ├── IMPLEMENTATION.md
│   │   └── TRADEOFFS.md
│   └── job-radar/                         <- Automated job discovery pipeline
│       ├── PROBLEMS.md
│       ├── IMPLEMENTATION.md
│       ├── CUSTOMIZATION.md
│       ├── SETUP.md
│       └── TRADEOFFS.md
├── docs/
│   ├── ARCHITECTURE.md                    <- Network & infrastructure deep dive
│   ├── SECURITY.md                        <- Security design & policy documentation
│   └── LESSONS.md                         <- Operational lessons learned
├── configs/                               <- Sanitized configuration files
│   ├── telegraf/
│   ├── grub/
│   └── vfio/
└── blog/
    ├── from-jamf-to-proxmox.md            <- Career transition writeup
    └── active-directory-hybrid-identity-lab.md <- AD lab build log
```

---

## Related

| Repo | Description |
|------|-------------|
| [technical-writeups](https://github.com/timanlemvo/technical-writeups) | Incident forensics, infrastructure hardening, and systems troubleshooting documentation |

---

**Contact:** [LinkedIn](https://linkedin.com/in/timanlemvo) · [Portfolio](https://tima.dev) · timanlemvo@gmail.com
