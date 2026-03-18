# Security Monitoring: Wazuh SIEM Fleet-Wide Detection Pipeline

Centralized threat detection and alerting across the entire Alliance Fleet, with real-time alerts routed to Discord.

## Overview

Wazuh runs as an all-in-one deployment (manager + indexer + dashboard) on LXC 110 on Node-C (Gozanti Cruiser). Ten agents across all three Proxmox hosts, VMs, and LXC containers ship security events to the manager, which evaluates them against built-in and custom detection rules. Alerts at level 5+ are routed through the Slack-compatible integration module to Discord's `/slack` webhook endpoint, landing in **#alert-triage** with severity color coding.

| Component | Location | Purpose |
|-----------|----------|---------|
| Wazuh Manager | LXC 110 · 192.168.20.30 · VLAN 20 | Rule evaluation, alert generation, dashboard |
| Wazuh Agents (×10) | All nodes, VMs, LXCs | Log collection, FIM, rootcheck |
| slack.py (modified) | `/var/ossec/integrations/` | Slack-compatible payload → Discord webhook |
| Discord #alert-triage | Alliance Fleet server | Alert triage with severity colors (green/yellow/red) |

## Key Constraints

- **Single point of failure**: Wazuh Manager on Node-C has no clustering or hot standby
- **Rootcheck false positives**: LXC `/dev/.lxc/*` paths trigger rule 510 on every container scan
- **Level 5+ threshold**: Informational events (level 3–4) only visible in the dashboard, not Discord
- **slack.py required patching**: `msg['ts'] = alert['id']` causes silent HTTP 400 from Discord, must be commented out
- **Manual install failed**: Debian 13 dependency conflicts required pivot to the community automation script

## Documentation

Read in order:

1. **[PROBLEM](PROBLEM.md)**  What was broken before Wazuh: zero centralized logging, blind spots exposed by the VFIO lockup incident, and why a single operator needs automated detection.
2. **[TRADEOFFS](TRADEOFFS.md)**  Why Wazuh over Splunk/ELK/cloud SIEM, rootcheck noise reality, Node-C SPOF, the seven-hour manual install lesson, and when to reconsider.
3. **[IMPLEMENTATION](IMPLEMENTATION.md)**  How it was built: architecture, agent enrollment (all 10), ossec.conf integration block, the slack.py fix, Discord pipeline setup, and test procedures.

## Agent Enrollment

| Host | Role | Node | Status |
|------|------|------|--------|
| FCM2250 | Proxmox hypervisor | Node-A (Falcon) | ✅ Enrolled |
| QCM1255 | Proxmox hypervisor | Node-B (Corvette) | ✅ Enrolled |
| OptiPlex7050 | Proxmox hypervisor | Node-C (Gozanti) | ✅ Enrolled |
| AdGuard LXC | DNS filtering | Node-C | ✅ Enrolled |
| Tantive-III (VM) | AI/ML workloads | Node-A | ✅ Enrolled |
| Home One (VM 200) | Authentik, Portainer, PostgreSQL, Redis | Node-C | ✅ Enrolled |
| Phoenix-Nest (VM 202) | n8n, Vaultwarden, Homepage, Uptime Kuma | Node-B | ✅ Enrolled |
| Stinger Mantis (VM 203) | BD-1 Personal Assistant | Node-B | ✅ Enrolled |
| NPM LXC (CT 101) | Nginx Proxy Manager (DMZ) | Node-C | ✅ Enrolled |
| InfluxDB LXC (111) | InfluxDB 2.x time-series DB | Node-B | ✅ Enrolled |
| Grafana LXC (112) | Grafana dashboards | Node-B | ⏳ Pending |

## Related Projects

- **[AI Platform](../ai-platform/)**: Tantive-III GPU inference stack with VFIO passthrough
- **[Identity & Access](../identity-access/)**: Authentik SSO zero-trust identity platform
