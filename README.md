# Proxmox Homelab: Cloud, Security & Platform Engineering Lab

Production-grade infrastructure running 25+ services across a 3-node cluster with zero-trust identity, security monitoring, full-stack observability, GPU-accelerated AI workloads, and hybrid Active Directory with Entra Connect sync.

---

## Overview

This is a working simulation of enterprise infrastructure, built by applying lessons from 7 years managing 200+ users at Team Liquid, Stagwell, and CAA. Every design decision (VLAN segmentation, centralized identity, observability pipelines, threat detection, hybrid identity) mirrors production standards implemented professionally.

`Linux Administration` `Proxmox VE Clustering` `Active Directory` `Hybrid Identity (Entra Connect)` `VLAN Segmentation` `Firewall Policy` `SIEM/XDR (Wazuh)` `SSO/IAM (Authentik)` `Observability (Telegraf → InfluxDB → Grafana)` `Zero-Trust (Tailscale)` `GPU Passthrough (VFIO/IOMMU)` `Reverse Proxy & TLS` `Threat Detection & Response` `Incident Forensics` `Infrastructure Documentation`

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        PROXMOX VE CLUSTER (3 Nodes)                          │
│                      Corosync Quorum │ Kernel 6.17.x-pve                     │
├─────────────────────────┬─────────────────────────┬──────────────────────────┤
│      NODE-A             │      NODE-B             │      NODE-C              │
│   "Millennium Falcon"   │    "CR90 Corvette"      │   "Gozanti Cruiser"      │
│      (FCM2250)          │      (QCM1255)          │    (OptiPlex 7050)       │
├─────────────────────────┼─────────────────────────┼──────────────────────────┤
│ Intel Core Ultra 9      │ AMD Ryzen 7 PRO         │ Intel i7-7700            │
│ 64GB DDR5               │ 64GB DDR5 ECC           │ 32GB DDR4                │
│ 2TB NVMe Gen4           │ 4TB Storage             │ 512GB NVMe + 1TB SATA    │
│ RTX 4000 Ada (20GB)     │                         │ 2.5GbE NIC (modded)      │
├─────────────────────────┼─────────────────────────┼──────────────────────────┤
│ ROLE: AI/ML Compute     │ ROLE: Data & Operations │ ROLE: Network & Security │
│                         │                         │                          │
│ ┌─────────────────────┐ │ ┌─────────────────────┐ │ ┌──────────────────────┐ │
│ │ Tantive-III VM      │ │ │ ALLIANCE-DC01       │ │ │ Wazuh SIEM           │ │
│ │ • Ollama LLM        │ │ │ (AD DS + DNS)       │ │ │ AdGuard DNS          │ │
│ │ • OpenWebUI         │ │ │ PostgreSQL          │ │ │ Nginx Proxy Mgr      │ │
│ │ • ComfyUI           │ │ │ n8n Automation      │ │ │ UptimeKuma           │ │
│ │ • AnythingLLM       │ │ │ Authentik SSO       │ │ └──────────────────────┘ │
│ └─────────────────────┘ │ │ InfluxDB            │ │                          │
│                         │ │ Grafana             │ │                          │
│                         │ │ Vaultwarden         │ │                          │
│                         │ │ HomeAssistant       │ │                          │
│                         │ │ Homepage            │ │                          │
│                         │ └─────────────────────┘ │                          │
└─────────────────────────┴─────────────────────────┴──────────────────────────┘
```
**[Live Architecture Diagram →](https://timanlemvo.github.io/Alliance-homelab-infrastructure/)**

### Hardware Specifications

| Node | Codename | Processor | Memory | Storage | Special Hardware |
|------|----------|-----------|--------|---------|------------------|
| Node-A | Millennium Falcon | Intel Core Ultra 9 | 64GB DDR5 | 2TB NVMe Gen4 | RTX 4000 Ada 20GB (VFIO passthrough) |
| Node-B | CR90 Corvette | AMD Ryzen 7 PRO | 64GB DDR5 ECC | 4TB | ECC memory for data integrity, hosts ALLIANCE-DC01 |
| Node-C | Gozanti Cruiser | Intel i7-7700 | 32GB DDR4 | 512GB NVMe + 1TB SATA | 2.5GbE NIC hardware mod |

Each node is purpose-built for its workload. Node-A carries the GPU for AI/ML inference. 20GB VRAM handles large parameter models. Node-B uses ECC memory because it runs the data pipeline where silent bit-flip corruption in time-series or authentication data would poison monitoring and identity. It also hosts ALLIANCE-DC01: ECC RAM directly protects NTDS.dit from silent memory corruption, making it the right home for the domain controller. Node-C handles network-edge services and serves as the Tailscale subnet router, keeping the security control plane on a dedicated node.

---

## Network Architecture

### VLAN Structure

```
╔══════════════════════════════════════════════════════════════════════════╗
║                            VLAN TOPOLOGY                                 ║
╠══════════════════════════════════════════════════════════════════════════╣
║                                                                          ║
║  VLAN 10 - MANAGEMENT / TATOOINE (192.168.1.0/24)                        ║
║  ├─ 192.168.1.1   UniFi Dream Machine (Gateway/Firewall)                 ║
║  ├─ 192.168.1.2   UniFi US-8-150W (Switch)                               ║
║  ├─ 192.168.1.10  Node-A Proxmox (Millennium Falcon)                     ║
║  ├─ 192.168.1.11  Node-B Proxmox (CR90 Corvette)                         ║
║  ├─ 192.168.1.12  Node-C Proxmox (Gozanti Cruiser)                       ║
║  ├─ 192.168.1.50  ALLIANCE-DC01 (Windows Server 2022, AD DS + DNS)       ║
║  └─ No DHCP, static assignments only                                     ║
║                                                                          ║
║  VLAN 20 - SERVICES / NABOO (192.168.20.0/24)                            ║
║  ├─ 192.168.20.10 Authentik SSO (PostgreSQL, Redis)                      ║
║  ├─ 192.168.20.20 Tantive-III VM (Ollama, OpenWebUI, ComfyUI)            ║
║  ├─ 192.168.20.30 Wazuh SIEM                                             ║
║  ├─ 192.168.20.40 Grafana                                                ║
║  ├─ 192.168.20.41 InfluxDB                                               ║
║  ├─ 192.168.20.50 n8n Automation                                         ║
║  ├─ 192.168.20.51 Vaultwarden                                            ║
║  ├─ 192.168.20.86 Canto-Bight (Windows 11 Pro, AD test workstation)      ║
║  └─ DHCP .100-.200 for future services                                   ║
║                                                                          ║
║  VLAN 30 - IoT / MUSTAFAR (192.168.30.0/24)                              ║
║  ├─ 192.168.30.10 HomeAssistant                                          ║
║  ├─ Smart home devices                                                   ║
║  └─ Isolated, cannot initiate to Management or Services                  ║
║                                                                          ║
║  VLAN 40 - DMZ / SCARIF (192.168.40.0/24)                                ║
║  ├─ 192.168.40.10 Nginx Proxy Manager                                    ║
║  ├─ Public-facing ingress only                                           ║
║  └─ No DHCP, static assignments only                                     ║
║                                                                          ║
║  VLAN 50 - BOT-NET / SECURITY SANDBOX (192.168.50.0/28)                  ║
║  ├─ 192.168.50.10 Fulcrum VM204 (K-2SO Security Observer)                ║
║  ├─ Triple isolation: no LAN, no internet, Discord webhooks only         ║
║  └─ No DHCP, static assignments only                                     ║
║                                                                          ║
╚══════════════════════════════════════════════════════════════════════════╝
```

| VLAN | Codename | Name | Subnet | Gateway | DHCP | Purpose |
|------|----------|------|--------|---------|------|---------|
| 10 | Tatooine | Management | 192.168.1.0/24 | 192.168.1.1 | Disabled | Hypervisor & infrastructure management |
| 20 | Naboo | Services | 192.168.20.0/24 | 192.168.20.1 | .100-.200 | Application workloads |
| 30 | Mustafar | IoT | 192.168.30.0/24 | 192.168.30.1 | .100-.200 | Smart home, fully isolated |
| 40 | Scarif | DMZ | 192.168.40.0/24 | 192.168.40.1 | Disabled | Public-facing reverse proxy |
| 50 | Bot-Net | Security Sandbox | 192.168.50.0/28 | 192.168.50.1 | Disabled | K-2SO triple isolation (Fulcrum VM204) |

DHCP is disabled on Management, DMZ, and Bot-Net. Management contains hypervisor interfaces. Unauthorized access means full infrastructure compromise. DMZ is the external attack surface. Bot-Net is a triple-isolated sandbox for K-2SO: no LAN access, no internet, Discord webhooks only. Every host on these VLANs must be explicitly provisioned.

IoT is fully isolated. Consumer IoT devices are notoriously insecure. VLAN 30 can reach the internet but cannot initiate connections to Management or Services.

### Network Hardware

| Device | Model | Role |
|--------|-------|------|
| Gateway/Firewall | UniFi Dream Machine | Routing, inter-VLAN firewall, VLAN management, threat management |
| Switch | UniFi US-8-150W | PoE managed switch, VLAN trunking to all nodes |
| WiFi AP | UniFi Beacon HD | Wireless connectivity |

Previously ran OPNsense as a dedicated firewall, but persistent issues with the ISP modem releasing IP addresses during the dual-gateway setup led to reliability problems. Consolidated to the UniFi Dream Machine as the single gateway/firewall. The tradeoff is less granular firewall API access. Automated threat response is being rebuilt for the UDM platform.

### Traffic Flow: External Request

```
Internet
  │
  ▼
UniFi Dream Machine (firewall, threat management, inter-VLAN routing)
  │
  ▼
VLAN 40 / Scarif --- Nginx Proxy Manager (TLS termination, rate limiting)
  │
  ▼
VLAN 20 / Naboo --- Authentik (SSO challenge, MFA, policy evaluation)
  │
  ▼
VLAN 20 / Naboo --- Backend Service (Grafana, n8n, OpenWebUI, etc.)
```

### Remote Access: Tailscale

```
┌─────────────────────────────────────────────────────────────┐
│                    TAILSCALE OVERLAY                        │
├─────────────────────────────────────────────────────────────┤
│  100.x.x.10 --> Node-A (Millennium Falcon)                  │
│  100.x.x.11 --> Node-B (CR90 Corvette)                      │
│  100.x.x.12 --> Node-C (Gozanti Cruiser) <- Subnet Router   │
│  100.x.x.20 --> Selected VMs/Containers                     │
└─────────────────────────────────────────────────────────────┘
```

Node-C advertises the management subnet to authorized Tailscale clients. Remote devices enrolled in the tailnet can reach infrastructure without any ports exposed to the public internet.

---

## Security Stack

```
Layer 1  - PERIMETER:    UniFi Dream Machine firewall + threat management
Layer 2  - NETWORK:      VLAN segmentation + inter-VLAN firewall rules
Layer 3  - ACCESS:       Tailscale zero-trust overlay (no exposed ports)
Layer 4  - IDENTITY:     Authentik SSO + MFA (TOTP/WebAuthn) across all services
Layer 5  - INGRESS:      Nginx Proxy Manager (TLS termination, rate limiting)
Layer 6  - DNS:          AdGuard filtering (ad/tracker/malware domain blocking)
Layer 7  - SECRETS:      Vaultwarden (self-hosted Bitwarden)
Layer 8  - DETECTION:    Wazuh SIEM/XDR (log aggregation, threat detection, FIM)
Layer 9  - ALERTING:     n8n automation (Wazuh alerts → Discord notifications)
Layer 10 - MONITORING:   Telegraf → InfluxDB → Grafana + UptimeKuma
```

---

## Implemented Capabilities

### Hybrid Active Directory & Identity

**Stack:** Windows Server 2022 (AD DS) → Entra Connect → Azure AD (Password Hash Sync)

Domain controller (`ALLIANCE-DC01`) built from scratch on Node-B (ECC RAM for NTDS.dit integrity). Forest `alliance.lab` with full OU structure, cascading GPO inheritance, and break/fix scenario testing. Entra Connect configured with Password Hash Sync syncing identities to an existing Azure tenant, closing the on-prem to cloud identity loop. Domain-joined Windows 11 Pro workstation (Canto-Bight) used for end-to-end GPO validation. Includes a full troubleshooting arc: Server Core vs Desktop Experience root cause diagnosis, VM rebuild decision, cross-VLAN domain join DNS fix, and Proxmox migration CD/DVD blocker resolution.

[Active Directory Hybrid Identity Lab →](projects/active-directory-lab/)

### Security Monitoring & Threat Detection

**Stack:** Wazuh SIEM/XDR → n8n → Discord alerting

Wazuh deployed with log aggregation, file integrity monitoring, and rule-based threat detection across the cluster. n8n processes Wazuh webhooks and pushes alerts to Discord for real-time visibility. Previously included automated IP blocking via OPNsense API. Automated enforcement is being rebuilt for the UniFi platform. Detection and alerting remain fully operational.

[SIEM Automation Pipeline →](projects/security-monitoring/)

### Zero-Trust Identity & Access Management

**Stack:** Authentik → OIDC/SAML → 15+ services, MFA enforced (TOTP + WebAuthn)

Single SSO login with 100% MFA coverage across all services. Complete login audit trail forwarded to Wazuh.

[Zero-Trust Identity Platform →](projects/identity-access/)

### Full-Stack Observability

**Stack:** Telegraf → InfluxDB 2.x (Flux) → Grafana → Discord alerts

10-second metric granularity across all 3 nodes. This pipeline was the sole reason a hard lockup root cause analysis was possible when all local logs were destroyed by `log2ram`.

[VFIO Lockup Forensics →](https://github.com/timanlemvo/technical-writeups/tree/main/proxmox-vfio-lockup-forensics)

### Network Segmentation

**Stack:** UniFi Dream Machine + 4 VLANs

Management, Services, IoT, and DMZ fully segmented. IoT devices cannot reach infrastructure. DMZ is the only public-facing surface. DHCP disabled on high-trust and high-exposure VLANs.

### GPU-Accelerated AI/ML Platform

**Stack:** Ollama + ComfyUI + AnythingLLM, RTX 4000 Ada via VFIO passthrough

Local LLM inference with no external API dependency. RAG pipeline for querying internal documents. GPU stable post-PCIe mitigation.

[GPU AI Platform →](projects/ai-platform/)

### Automation & Orchestration

**Stack:** n8n workflows + Docker Compose

Event-driven alerting, infrastructure notifications to Discord, HomeAssistant integration on isolated IoT VLAN.

---

## Projects

| Project | Problem Solved | Tech | Evidence |
|---------|----------------|------|----------|
| [Active Directory Hybrid Identity Lab](projects/active-directory-lab/) | No on-prem AD or hybrid identity in the fleet | Windows Server 2022, AD DS, GPO, Entra Connect, PHS | Domain controller, cascading GPOs, domain-joined Win11 workstation, Azure sync |
| [Zero-Trust Identity](projects/identity-access/) | Password sprawl, no MFA | Authentik OIDC/SAML | 15+ services, 100% MFA, full audit trail |
| [SIEM Pipeline](projects/security-monitoring/) | Manual threat monitoring doesn't scale | Wazuh + n8n + Discord | Automated detection and alerting, prior auto-blocking |
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
| Proxmox 3-node cluster | VMware / Hyper-V / Nutanix | Systems Engineer, Virtualization |
| Ollama + GPU passthrough | SageMaker / ML Platform | MLOps, AI Infrastructure |
| Tailscale zero-trust | Zscaler / Cloudflare Access | Cloud Security, Zero Trust |
| Incident forensics writeup | Production postmortem | SRE, Incident Response |

---

## Current State & Roadmap

### Implemented

- 3-node Proxmox cluster with corosync quorum
- 5-VLAN segmentation (Management/Tatooine, Services/Naboo, IoT/Mustafar, DMZ/Scarif, Security Sandbox/Bot-Net)
- Active Directory domain controller (ALLIANCE-DC01) on Node-B with cascading GPOs and break/fix testing
- Entra Connect hybrid identity sync (Password Hash Sync) to Azure tenant
- Centralized SSO via Authentik with MFA enforcement across 15+ services
- Wazuh SIEM deployed with detection rules, FIM, and log aggregation
- Full observability pipeline (Telegraf → InfluxDB → Grafana)
- Tailscale zero-trust remote access with subnet routing
- GPU passthrough with PCIe stability mitigations
- DNS filtering via AdGuard
- Reverse proxy with TLS termination via NPM
- Self-hosted password management (Vaultwarden)

### In Progress

- Automated threat response integration for UniFi Dream Machine (rebuilding after OPNsense migration)
- Wazuh agent tuning and custom detection rule expansion
- Grafana dashboard buildout for cluster-wide visibility
- Inter-VLAN firewall rule hardening
- Tailscale ACL policy refinement
- Second domain controller on Node-C for AD redundancy

### Planned

| Initiative | Why |
|-----------|-----|
| Proxmox Backup Server | Scheduled VM/CT snapshots with retention policies |
| Offsite encrypted backups | B2 or S3-compatible replication for disaster recovery |
| Kubernetes (k3s) cluster on Sentinel + Tydirium | Two Raspberry Pi 4B nodes dedicated to container orchestration — next portfolio milestone, in the pipeline |
| Terraform for VM provisioning | Full IaC, GitOps workflow |
| Grafana alerting rules | CPU, memory, disk, service availability thresholds |

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