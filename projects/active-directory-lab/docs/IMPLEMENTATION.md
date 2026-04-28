# Active Directory Hybrid Identity Lab
## IMPLEMENTATION.md

### Infrastructure

| Component | Details |
|---|---|
| Domain Controller | ALLIANCE-DC01, Windows Server 2022 (Desktop Experience) |
| DC IP | 192.168.1.50, VLAN 1 (Management) |
| Forest | alliance.lab |
| Node (final) | Node-B CR90 Corvette (QCM1255, AMD Ryzen 7 PRO, ECC RAM) |
| Test Workstation | Canto-Bight, Windows 11 Pro, VM 400 |
| Workstation IP | 192.168.20.86, VLAN 20 (Services) |
| Azure Tenant | alliance-fleet-rg, West US 2 |
| Sync Tool | Microsoft Entra Connect |

---

### Phase 1: Domain Controller Build

Created the VM on Millennium Falcon (Node-A) via Proxmox: 4 vCPU, 4GB RAM, 60GB disk, Intel E1000 NIC (VirtIO initially selected, no in-box driver in Windows Server, swapped to E1000 to resolve), VLAN 1.

Booted the Windows Server 2022 evaluation ISO and selected **Standard (Desktop Experience)**. This selection became critical later.

Configured via SConfig post-install: computer name set to `ALLIANCE-DC01`, static IP `192.168.1.50`, subnet `255.255.255.0`, gateway `192.168.1.1`, DNS self-pointed to `127.0.0.1` (standard for a DC that will become its own DNS server).

Dropped to PowerShell and installed the AD DS role:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "alliance.lab" -InstallDns:$true -Force:$true
```

Server rebooted into the promoted DC. Forest `alliance.lab` live.

---

### Phase 2: OU Structure, Users, and Groups

Built an OU hierarchy to mirror a real organization:

```
alliance.lab
└── Departments
    ├── IT
    ├── HR
    └── Finance
```

Created three test users (john.doe, jane.smith, bob.wilson) and placed them in their respective OUs. Created corresponding security groups (IT-Staff, HR-Staff, Finance-Staff) with members assigned.

All via PowerShell:

```powershell
New-ADOrganizationalUnit -Name "Departments" -Path "DC=alliance,DC=lab"
New-ADOrganizationalUnit -Name "IT" -Path "OU=Departments,DC=alliance,DC=lab"
New-ADUser -Name "John Doe" -SamAccountName "john.doe" -UserPrincipalName "john.doe@alliance.lab" -Path "OU=IT,OU=Departments,DC=alliance,DC=lab" -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) -Enabled $true
```

---

### Phase 3: Canto-Bight Build and Domain Join

VM 400 previously ran Ubuntu 25.04 as the fleet's gaming and streaming box. Destroyed the Ubuntu VM, cleaned the disks from vg-fast, and rebuilt as a Windows 11 Pro workstation for AD lab use.

VM spec:

| Setting | Value |
|---|---|
| VM ID | 400 |
| Name | Canto-Bight |
| Machine | q35 |
| BIOS | OVMF (UEFI) |
| TPM | v2.0 state disk on local-lvm (required for Windows 11) |
| CPU | 8 vCPU, host type |
| RAM | 16GB |
| Disk | 150GB, fast-lvm |
| NIC | VirtIO, vmbr0, VLAN tag 20 |
| IP | 192.168.20.86 |

Windows 11 Pro does not include VirtIO drivers in-box. Without them the installer cannot see the VirtIO disk or NIC. Fix: mount the VirtIO drivers ISO (Fedora stable release) as a second CD drive before first boot and load drivers from it when prompted during install. Installation completed clean.

Domain join:

```powershell
Add-Computer -DomainName "alliance.lab" -Credential alliance\Administrator -Restart
```

Verified in ADUC on DC-01: `CANTO-BIGHT` present in the Computers container under `alliance.lab`.

RDP enabled via Settings > System > Remote Desktop. Accessible at `192.168.20.86` or `Canto-Bight.alliance.lab` from any machine using DC-01 as DNS.

Snapshot `post-domain-join-clean` taken immediately: captured live RAM state (4.98GB), VirtIO disk, EFI disk, and TPM state. Clean restore point before any further configuration.

---

### Phase 4: GPO Cascade Validation

This is the core of what Canto-Bight was built to test. The goal was to verify GPO inheritance behavior across the OU structure, specifically how policies applied at the domain root cascade down through OUs and how OU-level GPOs interact with (and can override) higher-level ones.

Three GPOs configured:

**Password Policy GPO** (linked to domain root): Minimum length 12 characters, complexity enabled, lockout after 5 bad attempts. This cascades to all OUs by inheritance.

**Restrict Control Panel GPO** (linked to HR and Finance OUs): Blocks access to Control Panel and Settings. IT OU deliberately excluded to demonstrate OU-level scope.

**Drive Mapping GPO** (linked to Departments OU): Maps a network share on login. Tests that computer and user configuration settings apply correctly to domain-joined machines.

With Canto-Bight joined to `alliance.lab`, GPO application was verified using:

```powershell
gpupdate /force
gpresult /r
```

`gpresult /r` output confirmed correct policy inheritance: domain-level password policy applying to all users, Control Panel restriction applying to HR and Finance but not IT, drive mapping applying on Canto-Bight login.

Break/fix scenarios also tested: account lockout triggered via scripted bad password attempts, unlocked via `Unlock-ADAccount`, verified via `Search-ADAccount -LockedOut`. DNS misconfiguration introduced and resolved via DNS Manager. Computer account disable/re-enable cycle tested in ADUC.

---

### Phase 5: The Server Core Mistake and Rebuild

The first DC build used **Windows Server 2022 Standard** instead of **Standard (Desktop Experience)**. This wasn't caught until Day 4 when Entra Connect installation was attempted.

The Entra Connect setup wizard threw a `XamlParseException` immediately on launch. The installer's configuration UI is built on WPF/XAML, which requires Desktop Experience components that Server Core does not include.

Attempted fix: `Install-WindowsFeature Server-Gui-Shell, Server-Gui-Mgmt-Infra -Restart`. Returned `NameDoesNotExist`. Confirmed the issue by running:

```powershell
Get-WindowsFeature | Where-Object {$_.Name -like "*Gui*" -or $_.Name -like "*Desktop*"} | Select-Object Name, DisplayName, InstallState
```

Output: only `Remote-Desktop-Services` listed. Desktop Experience packages were not available because the evaluation ISO's Core edition doesn't include them. There was no upgrade path in place.

Decision: rebuild. The evaluation ISO does include both editions at the version selection screen during install. Wiped the VM, booted the ISO again, selected **Standard (Desktop Experience)** this time. Full rebuild including AD DS promotion, OU structure, users, groups, and GPOs took approximately 40 minutes given the familiarity from the first pass.

The rebuild was cleaner than the original. Knowing the VirtIO NIC issue in advance, the E1000 adapter was set before first boot. SConfig configuration was faster. PowerShell-first approach throughout meant no time lost clicking through Server Manager.

Lesson documented: always verify the Desktop Experience selection during Windows Server installation if GUI-dependent tools are planned. Server Core is the right choice for hardened production DCs, but it requires CLI-only tooling throughout the stack.

---

### Phase 6: Entra Connect and Hybrid Identity

With Desktop Experience installed, Entra Connect setup completed against the existing `alliance-fleet-rg` Azure tenant.

Prerequisites confirmed before install:

```powershell
[Net.ServicePointManager]::SecurityProtocol
$PSVersionTable.PSVersion
(Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full").Release
```

TLS 1.2 available, PowerShell 5.1, .NET 4.8. All clear.

Entra Connect installed and configured with Password Hash Sync. UPN alignment verified between on-premises `alliance.lab` accounts and the Azure tenant. Sync initiated, users from the `Departments` OU confirmed appearing in Entra ID.

This completes the hybrid identity loop: users exist in on-prem AD, sync to Entra ID, and can authenticate against both planes. The cascade GPO testing on Canto-Bight covers the on-premises policy layer. Entra ID covers the cloud identity layer.

---

### Phase 7: DC Migration to Node-B

After the lab was stable, `ALLIANCE-DC01` was migrated from Millennium Falcon (Node-A) to CR90 Corvette (Node-B) via Proxmox offline migration.

Rationale: Node-A hosts the RTX 4000 SFF Ada GPU via VFIO passthrough and has experienced two lockup incidents tied to PCIe/VFIO instability. Running a domain controller on the same node as volatile GPU passthrough workloads is poor practice. A DC going down takes DNS with it, which breaks every service in the fleet that uses the domain.

Node-B has ECC RAM (protects NTDS.dit from silent memory corruption) and is the designated ZFS data node. Both are directly relevant to DC reliability.

Migration steps: shut down VM, open Proxmox Migrate dialog for VM 300.

First attempt returned an immediate error: **Cannot migrate VM with local CD/DVD**. The Windows Server eval ISO was still mounted in the CD drive from the original install. Proxmox cannot migrate a local CD/DVD device since it is tied to the source node's storage.

Fix: Hardware tab on VM 300, select the CD/DVD drive, Edit, set media to "Do not use any media." Cleared the ISO, reopened the migrate dialog. Error gone. The remaining two yellow warnings about local disk migration time for the EFI disk (4MB) and system disk (60GB) are expected for non-shared-storage migrations and do not block the operation.

Migrated to Node-B (QCM1255), target storage fast-lvm. Verified boot on Node-B, confirmed DNS resolution and AD replication health post-move.

`ALLIANCE-DC01` now runs on the most stable node in the fleet.
