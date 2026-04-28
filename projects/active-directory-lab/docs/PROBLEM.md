# Active Directory Hybrid Identity Lab
## PROBLEM.md

### Background

Most IT roles at the mid-to-senior level expect hands-on Active Directory experience. Not just "I've used it" but the ability to build it, break it, fix it, and extend it into the cloud. After three years at Team Liquid managing Google Workspace and SaaS identity, my on-premises AD muscle had atrophied. An upcoming job interview required demonstrated AD administration experience: domain setup, GPO management, break/fix troubleshooting, and hybrid identity via Entra Connect.

The constraint was time. This had to be built, tested, and documented in two days.

---

### The Problem

No existing Windows Server infrastructure in the Alliance Fleet. Everything runs Linux-native across three Proxmox nodes, with identity handled by Authentik SSO. Active Directory, Group Policy, and hybrid Entra ID sync were entirely absent.

The goal: build a production-representative AD environment from scratch in a focused two-day sprint. Domain controller, OU structure with cascading GPO inheritance, a domain-joined Windows 11 workstation for end-to-end policy validation, and Entra Connect syncing identities to an existing Azure tenant.

---

### Why This Matters

Hybrid identity is the dominant model in enterprise IT. Most organizations run on-premises AD synced to Entra ID, with users authenticating against both planes. Understanding how those two systems interact, where sync conflicts happen, how GPOs cascade through an OU hierarchy, and how to troubleshoot the seam between on-prem and cloud is table stakes for Cloud, IAM, and Systems Engineering roles.

This lab covers that full stack under real time pressure, including a mid-sprint mistake that required a full VM rebuild and the troubleshooting process that led to that decision.

---

### Constraints

- Single DC (no redundancy): accepted lab limitation, documented as a known gap
- Windows Server 2022 Evaluation ISO: 180-day limit, sufficient for the use case
- Existing Azure tenant (`alliance-fleet-rg`, West US 2): already provisioned for homelab backups, repurposed as the Entra Connect sync target
- Node placement: initially Millennium Falcon (Node-A), migrated to CR90 Corvette (Node-B) post-sprint for ECC RAM and ZFS data integrity on NTDS.dit and SYSVOL
- Two-day sprint: all work from initial VM creation through Entra Connect sync and GPO validation completed across two consecutive days
