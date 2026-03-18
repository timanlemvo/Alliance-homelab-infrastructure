# PROBLEM: Security Monitoring (Wazuh SIEM)

## What Was Broken Before Wazuh

Before this deployment, security monitoring across the Alliance Fleet was reactive, checking logs manually after something felt off. The 3-node Proxmox cluster ran 25+ services across multiple VMs and LXC containers, but there was zero visibility into security events. No SIEM, no centralized log aggregation, no file integrity monitoring, no threat detection. Individual hosts logged locally, and some like Node-A running log2ram, stored journals entirely in RAM, meaning logs were destroyed on any unexpected reboot or failure.

The only operational telemetry came from the Telegraf → InfluxDB → Grafana pipeline, which tracked system performance metrics, not security events.

## Specific Security Blind Spots

The VFIO lockup incident on Node-A (February 9, 2026) proved how dangerous this gap was. The node suffered a complete hard lockup with zero local crash artifacts, no kernel panic, no pstore dump, no journal entries. The only reason root cause analysis was possible at all was because externally-stored InfluxDB telemetry provided a second-by-second timeline. If that performance pipeline hadn't existed, the incident would have been completely unrecoverable.

Beyond incident forensics, entire categories of security events were invisible: failed SSH attempts, file integrity changes on critical configs, privilege escalation, new user creation, and lateral movement between VLANs. Five high-traffic hosts, including the identity platform (Authentik), the automation engine (n8n/Vaultwarden), and the DMZ reverse proxy (NPM), were initially identified as HIGH-priority blind spots with no agent coverage at all.

## Why a Homelab Operator Needs SIEM

A single operator can't manually review logs across 10+ hosts. Without centralized detection and an alert pipeline that routes to where you actually work. In this case Discord security events don't exist. The constraint was clear: self-hosted, no cloud dependency, data stays in the cluster, and operable by one person.
