# Active Directory Hybrid Identity Lab

A two-day sprint to build a production-representative Active Directory environment from scratch, including cascading GPO testing, break/fix scenarios, Entra Connect hybrid identity sync, and a domain-joined Windows 11 workstation. Built under real time pressure as interview prep for a Cloud/Systems Engineering role.

---

## What This Covers

- Windows Server 2022 domain controller from scratch (forest promotion, DNS, OU structure)
- Group Policy cascade testing across a multi-OU hierarchy
- Break/fix scenarios: account lockout, DNS misconfiguration, computer account recovery
- Entra Connect Password Hash Sync to an existing Azure tenant
- Windows 11 Pro VM (Canto-Bight) built from scratch and domain-joined
- VirtIO driver installation for Windows VMs on Proxmox
- DC live migrations across nodes (Node-A, Node-B, then Node-D in October 2026)

Includes a documented rebuild after selecting the wrong Windows Server edition (Server Core vs. Desktop Experience), which blocked Entra Connect installation due to missing WPF/XAML rendering components.

---

## Infrastructure

| Component | Details |
|---|---|
| Domain Controller | ALLIANCE-DC01, Windows Server 2022 (Desktop Experience) |
| Forest | alliance.lab |
| DC IP | 192.168.1.50, VLAN 1 (Management) |
| Node (current) | Node-D GR-75 Transport (OptiPlex 7040 Micro), migrated from Node-B in October 2026 |
| Test Workstation | Canto-Bight, VM 400, Windows 11 Pro |
| Workstation IP | 192.168.20.86, VLAN 20 (Services) |
| Azure Tenant | alliance-fleet-rg, West US 2 |
| Sync | Microsoft Entra Connect, Password Hash Sync |

---

## Project Structure

```
active-directory-lab/
├── docs/
│   ├── PROBLEM.md          # Context, goals, and constraints
│   ├── IMPLEMENTATION.md   # Full technical walkthrough, all phases
│   └── TRADEOFFS.md        # Architecture decisions and documented gaps
├── blog/
│   └── blog-post.md        # Full write-up for holocron-labs.tima.dev
└── README.md
```

---

## Key Skills Demonstrated

**Active Directory administration:** Forest promotion, OU design, user and group management, GPO creation and link scoping, Group Policy inheritance and cascade behavior, break/fix troubleshooting.

**Hybrid identity:** Entra Connect configuration, Password Hash Sync, UPN alignment between on-premises AD and Entra ID.

**Windows Server:** Server 2022 deployment on Proxmox, SConfig configuration, PowerShell-first administration, Desktop Experience vs. Server Core tradeoffs.

**Proxmox:** Windows VM creation with q35/OVMF/TPM, VirtIO driver loading during install, VM migration between nodes, snapshot management.

**Troubleshooting:** Root cause diagnosis from XamlParseException to missing Desktop Experience packages, DNS resolution failure diagnosis and fix, VirtIO NIC driver gap, cross-VLAN domain join.

---

## Relevance to Cloud and IAM Roles

Hybrid identity (on-prem AD synced to Entra ID) is the dominant enterprise identity model. This lab covers the full stack: domain controller, Group Policy, Entra Connect sync, and a domain-joined endpoint for end-to-end validation. The Entra Connect configuration maps directly to SC-300 (Microsoft Identity and Access Administrator) exam objectives.

---

## Blog Post

Full write-up at [holocron-labs.tima.dev](https://holocron-labs.tima.dev) covering the complete sprint including the Server Core mistake, rebuild decision, and troubleshooting narrative.

---

## Related Projects

- [Zero-Trust Identity Platform](../identity-access/) — Authentik SSO (7 applications registered via OIDC and SAML, expansion in progress)
- [SIEM Automation Pipeline](../security-monitoring/) — Wazuh deployment with custom detection rules and n8n alert routing
