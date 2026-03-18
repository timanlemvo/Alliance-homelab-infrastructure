# TRADEOFFS: Security Monitoring (Wazuh SIEM)

## Why Wazuh

Wazuh won on three criteria that disqualified everything else at homelab scale.

**Over Splunk:** Cost. Splunk's free tier caps at 500MB/day with no alerting. The Alliance Fleet generates more than that during active attack periods. Splunk is the industry standard, but its per-GB pricing is designed for enterprises with security budgets, not single-operator homelabs.

**Over ELK Stack:** Operational overhead. ELK is more flexible for custom log parsing, but Wazuh ships with 3,000+ detection rules out of the box and native FIM. For a homelab where I'm not building a custom SIEM from scratch, Wazuh delivers more security value per hour invested.

**Over cloud SIEM (Sentinel, Datadog):** No cloud dependency. The entire detection and response pipeline runs on-prem. An internet outage shouldn't disable threat response, and homelab telemetry shouldn't leave the cluster.

**What tipped the scale:** Wazuh's Slack-compatible webhook integration slots directly into the n8n → Discord pipeline that already powers fleet operations. Detection to alert in under 3 seconds, landing in a channel I actually monitor.

## What Was Sacrificed

Wazuh doesn't have ML-driven correlation or behavioral baselining, every detection rule is hand-written. No native cloud integrations for AWS/Azure/GCP audit logs. No one-click compliance reporting. And the community, while growing, is significantly smaller than Splunk's or Elastic's. Troubleshooting edge cases sometimes means reading source code.

## Operational Pain

**Rootcheck false positives are the primary noise source.** LXC containers on Proxmox expose `/dev/.lxc/*` paths that Wazuh's rootcheck module flags as suspicious hidden files, every scan, every container, every time. The AdGuard agent alone fired rule 510 fifty-five times on the same benign LXC runtime file. Debian 13 (Trixie) also triggers generic trojaned-binary signatures on legitimate system utilities like `/bin/chsh` and `/bin/passwd`. These require custom suppression rules in `local_rules.xml` to silence without losing real detections.

**Node-C is a single point of failure.** The Wazuh Manager on LXC 110 (Gozanti Cruiser) has no clustering, no hot standby. If Node-C goes down, all security monitoring goes dark. Acceptable at homelab scale, but it means the security stack has the same HA gap as the services it's protecting.

**Alert threshold is a judgment call.** Level 5+ was chosen for the Discord integration to balance signal and noise. Level 3–4 events (informational but potentially relevant) are only visible in the Wazuh dashboard, which realistically gets checked less often than Discord.

## The Installation Lesson

The manual install path was a trap. Seven hours across two attempts component-by-component on Debian 13 produced a partially functional stack that broke on restart. The indexer worked, the manager sort-of worked, and the dashboard never got installed. Debian 13 (Trixie) wasn't officially supported, and the dependency tree had just enough mismatches to make the manual path unreliable.

The pivot: destroy the broken container, create a fresh one, run the community automation script. Twenty minutes later, every component was up. The script pins package versions, handles certificate generation, configures inter-component auth, and tests health at each step.

The manual attempts weren't wasted they taught me how the manager, indexer, and dashboard communicate via mutual TLS, where the config files live, and what breaks. But the goal was a working SIEM, not a PhD in Wazuh packaging. Know when to stop fighting and use the script.

## When I'd Reconsider

At 50+ agents, Wazuh's single-manager architecture starts to strain a distributed deployment with separate indexer nodes or a move to Elastic Security would be justified. If the fleet ever extends into cloud infrastructure, the lack of native CloudTrail/Azure Activity Log connectors becomes a real gap rather than a theoretical one. And if false positives ever exceed tolerance despite rule tuning, the alert noise reduction options are already scoped: raise the threshold to level 7 or filter to the six high-value rule IDs (SSH brute force, PAM failures, user creation, FIM changes).
