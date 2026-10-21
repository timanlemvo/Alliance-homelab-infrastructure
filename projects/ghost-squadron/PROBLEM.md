# Problem: Shared Fault Domains in the Fleet

## What Was Wrong

The Alliance Fleet runs on four Proxmox nodes, and several services the whole fleet depends on each live on a single node:

| Service | Lives on | What happens when that node is down |
|---------|----------|-------------------------------------|
| AdGuard Home (DNS) and Nginx Proxy Manager | Node-C | The UDM's Force DNS rule redirects every port 53 query to a dead resolver, so the whole fleet loses DNS |
| n8n (Wazuh to Discord alert path) | Node-B | Wazuh alerts have no route to Discord |
| Uptime Kuma, Grafana | Node-B | No uptime checks, no status page, no dashboards, and no way to ask "is the fleet even up?" |

Everything above also shares one hypervisor platform. A Proxmox-level problem (a bad upgrade, a cluster-wide issue) takes out the services and the monitoring that would report it.

## Requirements

- Independent hardware, so a Proxmox node failure cannot take it down
- A separate network segment, so it stays reachable and contained
- A different orchestrator, so a Proxmox-level problem does not reach it
- A small footprint: secondary services only, not a second fleet
- Default-deny toward the fleet's internal networks, with explicit holes only where needed

## Out of Scope

- A secondary reverse proxy. Running a second NPM needs certificate management, upstream configuration, and DNS failover logic beyond this project. Internal services stay reachable by IP when NPM is down, but external access still depends on Node-C and the Cloudflare tunnel.
- Replacing any primary service. Ghost Squadron is a fallback, not a migration.
