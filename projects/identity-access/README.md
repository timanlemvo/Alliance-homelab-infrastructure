# Identity & Access: Authentik SSO Zero-Trust Identity Platform

Centralized authentication with enforced MFA across 15+ services in the Alliance Fleet.

## Overview

Authentik runs on Home One (VM 200) on Node-C as a Docker Compose stack with PostgreSQL and Redis co-located to prevent split-brain identity failures. It serves as the single identity provider for the fleet, replacing 12+ scattered service-local logins with one SSO credential. MFA is enforced at the Authentik flow level, not per-app, so every integrated service inherits TOTP/WebAuthn challenges automatically. Services that don't natively support OIDC/SAML sit behind NPM with Authentik's forward auth outpost.

| Component | Location | Purpose |
|-----------|----------|---------|
| authentik-server | Home One · 192.168.20.10:9000 | Web UI, API, OIDC/SAML provider endpoints |
| authentik-worker | Home One · 192.168.20.10 | Background tasks, outpost management |
| PostgreSQL 15 | Home One · co-located | User directory, flow config, audit logs |
| Redis | Home One · co-located | Session storage, caching |
| Portainer | Home One · 192.168.20.10:9443 | Container management UI |

## Key Constraints

- **Single Authentik instance**: No High-Availablity, if Home One goes down, auth for all services fails
- **OIDC preferred over LDAP**: Every service in the stack supports OAuth2/OIDC natively; LDAP adds maintenance without benefit at this scale
- **Grafana `root_url` gotcha**: OAuth redirect defaults to `localhost` unless `[server]` section is explicitly configured, misdiagnosed as a provider misconfiguration
- **Forward auth for non-SSO services**: NPM outpost catches unauthenticated requests before they reach the application

## Documentation

Read in order:

1. **[PROBLEM](PROBLEM.md)**  The credential sprawl problem: 15+ services with independent logins, ~30% MFA coverage, no audit trail, no offboarding path.
2. **[TRADEOFFS](TRADEOFFS.md)**  Why Authentik over Keycloak/Authelia/Okta, OIDC over LDAP, what broke (redirect loops, MFA lockouts, session timeouts), and accepted limitations.
3. **[IMPLEMENTATION](IMPLEMENTATION.md)**  How it was built: Docker Compose deployment, OIDC/SAML provider setup, per-service integration configs, flow-level MFA enforcement, and the Grafana OAuth fix.

## Integrated Services

| Service | Protocol | MFA Enforced |
|---------|----------|:------------:|
| Proxmox | OIDC | ✅ |
| Portainer | OAuth2 | ✅ |
| Grafana | OIDC | ✅ |
| n8n | OIDC | ✅ |
| Vaultwarden | OIDC | ✅ |
| OpenWebUI | OAuth2 | ✅ |
| Wazuh Dashboard | SAML | ✅ |
| HomeAssistant | OIDC | ⚠️ Optional |
| Non-SSO services | NPM forward auth | ✅ |

## Related Projects

- **[Security Monitoring](../security-monitoring/)**: Wazuh SIEM with Wazuh-to-Discord alert pipeline
- **[AI Platform](../ai-platform/)**: Tantive-III GPU inference stack with VFIO passthrough
