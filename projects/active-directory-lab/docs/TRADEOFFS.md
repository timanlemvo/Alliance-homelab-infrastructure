# Active Directory Hybrid Identity Lab
## TRADEOFFS.md

### Single DC vs. Redundant DCs

A production AD environment always runs at least two domain controllers for redundancy. This lab runs one. If `ALLIANCE-DC01` goes down, DNS fails for the entire `alliance.lab` domain, which means every service in the fleet that relies on domain resolution stops resolving.

This is an accepted tradeoff for a lab. Adding a second DC on Node-C is on the roadmap. For now, the DC migration to Node-B (ECC RAM, ZFS, no GPU passthrough volatility) mitigates the risk as much as single-DC architecture allows.

---

### Server Core vs. Desktop Experience

Server Core is the right answer for a production DC. Smaller attack surface, lower memory footprint, fewer services running. Microsoft's own guidance pushes Server Core for DCs specifically.

This lab runs Desktop Experience because Entra Connect's setup wizard requires it. That's a real constraint in enterprise environments too: some Microsoft tooling still requires a GUI. The tradeoff was made deliberately and documented, not by accident.

Long-term: once Entra Connect is configured and stable, the DC could be rebuilt as Server Core with Entra Connect installed on a separate server or member server. That's the cleaner architecture.

---

### Entra Connect Password Hash Sync vs. Pass-Through Authentication

Password Hash Sync (PHS) was chosen over Pass-Through Authentication (PTA) for this lab. PHS syncs a hash of the password hash to Entra ID, meaning cloud authentication works even if the on-premises DC is unreachable. PTA requires the on-prem DC to be available for every authentication attempt.

For a homelab where the DC is a single VM that could be shut down for maintenance, PHS is the more resilient choice. In a production environment with redundant DCs and high availability requirements, PTA or federation (AD FS) might be preferred for tighter control over the authentication flow.

---

### Node-A vs. Node-B for the DC

The DC was initially built on Millennium Falcon (Node-A) because it had the most available compute at the time. The right answer was Node-B from the start. Node-A's VFIO passthrough history (two lockup incidents, GRUB hardening required, ComfyUI crash-loop events) makes it the wrong home for a service that owns DNS for the entire fleet.

The migration to Node-B was straightforward but added time. Better node selection upfront would have avoided it. That said, documenting the reasoning for the migration is itself useful: it demonstrates understanding of why DC placement matters in a real environment.

---

### Management VLAN vs. Services VLAN Placement

`ALLIANCE-DC01` sits on VLAN 1 (Management, 192.168.1.0/24). Canto-Bight sits on VLAN 20 (Services, 192.168.20.0/24). Cross-VLAN domain join works because UniFi firewall rules permit the necessary ports (Kerberos 88, LDAP 389, DNS 53, RPC) between the two VLANs.

A stricter segmentation approach would put the DC on a dedicated identity VLAN. For this lab, Management VLAN placement is fine and keeps the DC accessible from all management infrastructure without additional firewall complexity. In a production environment with tighter segmentation requirements, a dedicated identity VLAN with explicit allow rules would be the right call.

---

### What This Lab Doesn't Cover

- AD FS (federation): out of scope, PHS covers the hybrid auth use case for this environment
- RODC (read-only domain controller): relevant for branch office scenarios, not applicable here
- AD replication between multiple DCs: single DC limitation
- Fine-grained password policies: the lab uses a single domain-level password policy
- Privileged Access Workstations (PAW): relevant for production hardening, not implemented

These are documented gaps, not oversights. Each represents a real production consideration that could be added to extend the lab.
