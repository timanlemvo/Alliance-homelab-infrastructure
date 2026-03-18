

## 1. Architecture Overview

Wazuh Manager runs as an all-in-one deployment (manager + indexer + dashboard) on LXC 110, `192.168.20.30`, VLAN 20 (Services), hosted on Node-C (Gozanti Cruiser / OptiPlex 7050). Agents connect back on port 1514 (event data) and port 1515 (enrollment). The reusable deployment script is saved at `/tmp/install-agent.sh` on OptiPlex7050.

The alert pipeline flows from agents through the manager's rule engine into the Slack-compatible integration module, which posts to Discord's `/slack` webhook endpoint. The full chain:

```
/var/ossec/etc/ossec.conf
  └─► <integration> block
        ├── hook_url: Discord /slack webhook URL
        ├── level: 5
        └── calls /var/ossec/integrations/slack.py

slack.py (MODIFIED see Section 5)
  └─► Builds Slack-compatible JSON payload
  └─► Sends to Discord /slack compatibility endpoint
  └─► Discord renders as embed in #alert-triage
```

Alerts arrive in `#alert-triage` with severity color mapping: green (level ≤4), yellow (5–7), red (8+), and include MITRE ATT&CK technique tagging in the embeds. Alert types flowing include rootcheck, syscheck/FIM, user creation/deletion, PAM failures, and SSH events.

## 2. Agent Enrollment Status

**10 active agents, 1 pending (Grafana only).**

| Host | Role | Node | Status |
|------|------|------|--------|
| FCM2250 | Proxmox hypervisor | Node-A (Falcon) | ✅ Enrolled |
| QCM1255 | Proxmox hypervisor | Node-B (Corvette) | ✅ Enrolled |
| OptiPlex7050 | Proxmox hypervisor | Node-C (Gozanti) | ✅ Enrolled |
| AdGuard LXC | DNS filtering | Node-C (Gozanti) | ✅ Enrolled |
| Tantive-III (VM) | AI/ML workloads | Node-A (Falcon) | ✅ Enrolled |
| Home One (VM 200) | Authentik, Portainer, PostgreSQL, Redis | Node-C | ✅ Enrolled |
| Phoenix-Nest (VM 202) | n8n, Vaultwarden, Homepage, Uptime Kuma | Node-B | ✅ Enrolled |
| Stinger Mantis (VM 203) | BD-1 Personal Assistant | Node-B | ✅ Enrolled (v4.14.2) |
| NPM LXC (CT 101) | Nginx Proxy Manager (DMZ) | Node-C | ✅ Enrolled (v4.14.2) |
| InfluxDB LXC (111) | InfluxDB 2.x time-series DB | Node-B | ✅ Enrolled (v4.14.2) |
| Grafana LXC (112) | Grafana dashboards | Node-B | ⏳ Pending (MEDIUM) |
| Wazuh Server (LXC 110) | Manager/Indexer/Dashboard | Node-C | ❌ Excluded as circular dependency |

### Agent Deployment Process

SSH into the target host and run the reusable script:

```bash
bash /tmp/install-agent.sh "HostnameHere"
```

The script installs the DEB amd64 agent package (v4.14.2), sets the manager IP to `192.168.20.30`, enables and starts the service. After installation, if the agent shows a configuration error on start, the manager address likely wasn't set fix with:

```bash
sed -i 's|<address>.*</address>|<address>192.168.20.30</address>|' /var/ossec/etc/ossec.conf
systemctl restart wazuh-agent
```

On minimal LXC containers (like InfluxDB and NPM), the `lsb-release` dependency may be missing. Install it before configuring the agent:

```bash
apt update && apt install -y lsb-release
dpkg --configure wazuh-agent
```

Verify enrollment in the Wazuh Dashboard at `https://wazuh.tima.dev` under Endpoints.

## 3. ossec.conf Integration Block

The Discord integration is configured in `/var/ossec/etc/ossec.conf` on the Wazuh manager:

```xml
<integration>
  <n>slack</n>
  <hook_url>DISCORD_WEBHOOK_URL/slack</hook_url>
  <level>5</level>
  <alert_format>json</alert_format>
</integration>
```

**Critical note: No `<group>` tag.** Wazuh's internal group names (like `syslog`, `authentication_failed`) don't match `default`, so adding a `<group>` filter silently drops every alert. This was identified during the initial troubleshooting, integratord was running, alerts were firing at level 7+, but nothing reached Discord because the group filter was rejecting everything.

After any `ossec.conf` change:

```bash
systemctl restart wazuh-manager
```

## 4. Discord Integration Steps

The webhook URL must end with `/slack` to use Discord's Slack-compatible endpoint. The integration module (`integratord`) starts with the manager and processes alerts through `/var/ossec/integrations/slack.py`, which builds a Slack-compatible JSON payload with the `attachments` format.

To verify the pipeline is working end-to-end:

1. Confirm integratord is running: `ps aux | grep integratord`
2. Check integration log for startup: `grep -i integrat /var/ossec/logs/ossec.log | tail -20` should show `Enabling integration for: 'slack'`
3. Verify alerts are being generated above threshold: `tail -20 /var/ossec/logs/alerts/alerts.log`
4. If alerts exist in the log but don't appear in Discord, check for the `<group>` tag issue or the `slack.py` bug below.

## 5. Critical Fix: slack.py (March 11, 2026)

**Problem:** Discord's `/slack` compatibility endpoint rejected Wazuh alert payloads with HTTP 400, but the error was silent, no entries in integration logs unless debug mode was enabled manually.

**Root cause:** In `/var/ossec/integrations/slack.py`, the line `msg['ts'] = alert['id']` passes the Wazuh alert ID (e.g., `1773279793.212108`) as a Slack timestamp field. Discord's `/slack` endpoint is stricter than actual Slack about the `ts` format and rejects the entire payload.

**Diagnosis:** Running the script manually with debug output revealed the issue:

```bash
/var/ossec/framework/python/bin/python3 /var/ossec/integrations/slack.py \
  /tmp/test-alert.json "" "DISCORD_WEBHOOK_URL/slack" debug 2>&1
```

Output showed `Response [400]` with the payload containing `"ts": "1773279793.212108"`.

**Fix:**

```bash
cp /var/ossec/integrations/slack.py /var/ossec/integrations/slack.py.bak
nano /var/ossec/integrations/slack.py
# Comment out: msg['ts'] = alert['id']
systemctl restart wazuh-manager
```

**Backup preserved at:** `/var/ossec/integrations/slack.py.bak`

After applying this fix, alerts began flowing immediately: rootcheck events, user creation (rule 5902, level 8), PAM failures, all rendering in `#alert-triage` with proper severity color coding.

## 6. Test Procedure

Force a level 7+ alert by creating and deleting a test user:

```bash
sudo useradd wazuh_test && sudo userdel wazuh_test
```

This fires rule 5902 (level 8: "New user added to the system"), which is well above the level 5 threshold. The alert should appear in Discord `#alert-triage` within 30 seconds.

If testing the full pipeline from scratch, also verify:

```bash
# Confirm integratord is running
ps aux | grep integratord

# Watch alerts in real time
tail -f /var/ossec/logs/alerts/alerts.json

# Watch integration log
tail -f /var/ossec/logs/integrations.log
```

**Known noise source:** LXC rootcheck alerts (`/dev/.lxc/*` detections) are false positives from Proxmox container filesystems. When alert volume becomes unmanageable, apply threshold adjustment (`<level>7</level>`) or rule-based filtering (`<rule_id>5710,5712,5503,5504,5902,550</rule_id>`) in the integration block.
