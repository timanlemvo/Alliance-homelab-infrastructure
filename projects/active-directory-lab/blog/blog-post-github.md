# Building an Active Directory Lab from Scratch (And the Mistake That Made It Better)

I needed to build an Active Directory environment fast. Not because I wanted to, exactly, but because I had a job interview coming up and my on-prem AD muscle had gone soft after three years managing Google Workspace and SaaS identity at Team Liquid. The interview was Wednesday. It was Sunday.

This post covers what I built across two days, the mistake I made on day one that forced a full rebuild, and how the whole thing ended up as a working hybrid identity lab with cascading GPOs and Entra Connect syncing to Azure. It's part homelab build log, part honest account of what goes wrong when you're moving fast under pressure.

---

## The Setup

The Alliance Fleet doesn't run any Windows infrastructure natively. Everything is Linux: Proxmox on three nodes, Docker Compose services, Authentik for SSO, n8n for automation. Active Directory was a gap.

The goal was to close that gap in a single sprint: domain controller from scratch, OU structure with cascading GPOs, a domain-joined Windows 11 workstation for testing, and Entra Connect syncing identities to an existing Azure tenant. Full hybrid identity stack, documented and working, inside two days.

I had a Windows Server 2022 evaluation ISO, an existing Azure tenant already provisioned for homelab backups, and a Proxmox VM slot on Millennium Falcon (Node-A). Let's go.

---

## Standing Up the DC

Created the VM in Proxmox: 4 vCPU, 4GB RAM, 60GB disk. First hiccup came immediately. Set the NIC to VirtIO, booted Windows Server, and the network adapter didn't exist. Windows Server has no in-box VirtIO drivers. Changed the NIC model to Intel E1000 in Proxmox, rebooted, adapter appeared. Noted for next time.

Configured the basics via SConfig: hostname `ALLIANCE-DC01`, static IP `192.168.1.50`, subnet `255.255.255.0`, gateway `192.168.1.1`, DNS self-pointed to `127.0.0.1`. Then dropped to PowerShell for the AD DS install:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "alliance.lab" -InstallDns:$true -Force:$true
```

Server rebooted into a promoted domain controller. Forest `alliance.lab` was live.

---

## OU Structure, Users, and GPOs

Built an OU hierarchy that mirrors a small organization:

```
alliance.lab
└── Departments
    ├── IT
    ├── HR
    └── Finance
```

Three test users, three security groups, everything placed in the correct OUs via PowerShell. Then the GPO work. Three policies configured:

A **Password Policy** linked at the domain root, enforcing 12-character minimum, complexity enabled, lockout after 5 failed attempts. This cascades to every OU by inheritance.

A **Restrict Control Panel** GPO linked to the HR and Finance OUs only, IT deliberately excluded. The kind of scoped policy you'd apply in a real environment where some users need system access and others don't.

A **Drive Mapping** GPO linked to the Departments OU, mapping a network share on login. Tests that user configuration policies apply correctly to domain-joined machines.

---

## Breaking Things on Purpose

With a working domain, the next step was intentionally breaking it. Triggered an account lockout by scripting six bad password attempts against a test user via SMB auth:

```powershell
$domain = "alliance.lab"
$username = "john.doe"
1..6 | ForEach-Object {
    & net use \\alliance-dc01\SYSVOL /user:$domain\$username "WrongPass$_" 2>$null
    & net use \\alliance-dc01\SYSVOL /delete 2>$null
}
```

`Get-ADUser john.doe -Properties LockedOut, BadLogonCount` confirmed `LockedOut: True`. Unlocked via `Unlock-ADAccount`, verified clean. Also introduced a DNS misconfiguration, disabled and re-enabled a computer account, and tested AD Recycle Bin restore. Real scenarios, muscle memory built.

---

## The Mistake

This is where day one got interesting.

Downloaded the Entra Connect installer, ran it on the DC:

```powershell
Start-Process "C:\Users\Administrator\AzureADConnectV2.msi"
```

The setup wizard opened and immediately crashed with a `XamlParseException`.

First instinct was a .NET version issue or a missing dependency. Spent time going down that path before landing on the real cause: the DC was running **Windows Server 2022 Standard**, the Server Core edition, not Desktop Experience. The Entra Connect setup wizard is built on WPF/XAML. Server Core doesn't have those rendering components. The wizard cannot run, full stop.

Attempted to add Desktop Experience after the fact, which returned `NameDoesNotExist`. Confirmed the actual available features:

```powershell
Get-WindowsFeature | Where-Object {$_.Name -like "*Gui*" -or $_.Name -like "*Desktop*"} | Select-Object Name, DisplayName, InstallState
```

Only `Remote-Desktop-Services` came back. The Core-only ISO edition doesn't include the Desktop Experience packages at all. No upgrade path without a reinstall.

The decision was quick: rebuild. The evaluation ISO includes both editions at the version selection screen. I had picked the wrong one at install and hadn't noticed because everything else worked fine until this exact moment.

Wiped the VM, booted the ISO, selected **Standard (Desktop Experience)** this time. The rebuild took about 40 minutes, faster than the original because every step was already familiar. The VirtIO NIC issue didn't catch me twice. SConfig went faster. PowerShell-first throughout.

The rebuild was honestly a cleaner build than the first one. The mistake forced a better pass.

---

## Day Two: Entra Connect, Canto-Bight, and Closing the Loop

With a DC running Desktop Experience, Entra Connect configuration completed cleanly.

Prerequisites confirmed first: TLS 1.2 available, PowerShell 5.1, .NET 4.8. All green. Installed Entra Connect, configured Password Hash Sync against the existing Azure tenant. PHS means cloud authentication works even if the on-prem DC is unreachable, which matters in a homelab where the DC is a single VM. UPN alignment verified between `alliance.lab` accounts and the Azure tenant. Sync initiated, users from the Departments OU confirmed in Entra ID.

The hybrid identity loop was closed: users in on-prem AD, synced to Entra ID, authenticating against both planes.

---

## Canto-Bight: The GPO Test Workstation

To actually validate GPO cascade behavior end to end, I needed a domain-joined Windows client. VM 400 had been running Ubuntu 25.04 as the fleet's gaming and streaming box. That got destroyed and rebuilt as a Windows 11 Pro workstation for the AD lab.

First, cleaned the old Ubuntu disks from vg-fast in Proxmox and rebuilt the VM with the right spec for Windows 11: q35 machine type, OVMF UEFI, TPM v2.0 state disk (Windows 11 requires TPM, Proxmox handles this as a software TPM state disk), 8 vCPU, 16GB RAM, 150GB on fast-lvm, VLAN 20.

One thing that catches people here: Windows 11 won't see the VirtIO disk or NIC during install without drivers. The installer just shows an empty disk list with a "Hardware not showing up? Load driver" link. The fix is to mount the VirtIO drivers ISO as a second CD drive before booting the installer, then load the Red Hat VirtIO SCSI pass-through driver from `D:\vioscs\w11\amd64` when prompted. Installation goes clean from there.

With Windows 11 Pro installed and on the network, the domain join required one extra step: DNS. Canto-Bight was pointing to AdGuard Home at `192.168.1.4` by default, which doesn't know about `alliance.lab`. Fixed by pointing DNS at the DC directly:

```powershell
netsh interface ip set dns "Ethernet" static 192.168.1.50
nslookup alliance.lab
```

`nslookup alliance.lab` resolving to `192.168.1.50` confirmed DNS was working. Then domain join:

```powershell
Add-Computer -DomainName "alliance.lab" -Credential alliance\Administrator -Restart
```

Verified in ADUC on DC-01. `CANTO-BIGHT` appeared in the Computers container under `alliance.lab`. RDP enabled, snapshot `post-domain-join-clean` taken immediately.

With Canto-Bight on the domain, GPO validation ran via `gpresult /r`. Password policy cascading from domain root: confirmed. Control Panel restriction applying to HR and Finance users but not IT: confirmed. Drive mapping on login: confirmed. That's the cascade working exactly as designed, end to end from DC to domain-joined workstation.

---

## Moving the DC to Node-B

After everything was stable, `ALLIANCE-DC01` was migrated from Millennium Falcon (Node-A) to CR90 Corvette (Node-B).

Node-A is the fleet's AI and ML workhorse. It runs an RTX 4000 SFF Ada via VFIO passthrough and has had two GPU-related lockup incidents that required GRUB parameter hardening to stabilize. Running a domain controller on the same node as volatile GPU passthrough workloads is a bad idea. A DC going down takes DNS with it, and when DNS goes down, everything in the fleet that depends on name resolution stops working at once.

Node-B has ECC RAM, protecting NTDS.dit from silent memory corruption. It's the designated ZFS data node and the most stable in the fleet. That's where the DC belongs.

First attempt at the migration hit an immediate wall: **Cannot migrate VM with local CD/DVD**. The Windows Server eval ISO was still mounted in the VM's CD drive. Proxmox can't migrate a local CD/DVD device because it's tied to the source node's storage.

Fix: Hardware tab on VM 300, select the CD/DVD drive, Edit, set to "Do not use any media." Cleared the ISO, opened the migrate dialog again, error gone. The two yellow warnings about local disk migration time for the EFI disk and system disk are expected for non-shared-storage migrations and don't block the operation.

Shut down the VM, migrated to Node-B (QCM1255), target storage fast-lvm. Verified boot on Node-B, confirmed DNS resolution and AD replication health post-move. `ALLIANCE-DC01` now runs on the most stable node in the fleet.

---

## What I Learned

The Server Core mistake was a real lesson in slowing down at the installer screen. It's an obvious choice in retrospect, four clearly labeled options, but when you're moving fast everything looks fine until the moment it doesn't. The fix was fast. The cost was about 40 minutes of rebuild time plus the time spent diagnosing before I found the root cause.

The troubleshooting process mattered more than the fix itself. Not just "it broke, rebuild it," but diagnosing the actual cause: `XamlParseException` to WPF/XAML dependency to missing Desktop Experience packages to no upgrade path available. Understanding why something can't be fixed in place, and making a deliberate decision to rebuild rather than keep poking at it, is a judgment call that comes up constantly in real infrastructure work.

The GPO cascade work reinforced something I knew conceptually but hadn't tested hands-on in a while: Group Policy inheritance is powerful and easy to get wrong. Domain-level policies cascade down through every OU. OU-level policies apply only where they're linked. The interaction between the two is where debugging gets interesting. `gpresult /r` is your best friend.

And Entra Connect closed the story on hybrid identity. On-prem AD and cloud Entra ID aren't separate systems in most enterprise environments. They're two planes of the same identity platform, and understanding the sync layer between them is table stakes for anything cloud or IAM adjacent.

---

## What's Next

A second DC on Node-C for redundancy is the obvious gap to close. Single DC is fine for a lab but it's a known risk worth documenting.

Longer term, rebuilding the DC as Server Core with Entra Connect running on a separate member server would be the cleaner production architecture. Desktop Experience on a DC was a necessary tradeoff to get the wizard running, not the right permanent answer.

Full documentation including implementation details and architecture tradeoffs lives in the [Alliance-Homelab-Infrastructure](https://github.com/timanlemvo/Alliance-Homelab-Infrastructure) repo.
