# PROBLEM: Zero-Trust Identity Platform (Authentik SSO)

## The Credential Sprawl Problem

Before Authentik, every service in the Alliance Fleet had its own login. Proxmox, Grafana, Portainer, n8n, Vaultwarden, OpenWebUI, Wazuh, HomeAssistant 15+ services, each with a unique username and password. Twelve or more credentials for a single operator. The same problems I'd seen managing Active Directory for 200+ users at CAA were reproducing at homelab scale: password reuse, inconsistent MFA enforcement, and zero visibility into which services were actually being accessed.

## What Was Missing

**No central authentication.** Adding a new service meant creating another local account with another password. There was no SSO each login was independent, each session was isolated.

**No MFA enforcement.** MFA coverage sat around 30%, applied manually per-app where the service happened to support it. There was no way to enforce MFA as a policy across the fleet.

**No audit trail.** Login events weren't aggregated. There was no way to answer "who accessed what, when" without checking each service's local logs individually if the service even logged authentication events at all.

**No offboarding path.** Revoking access to everything meant manually visiting each service and disabling or deleting accounts. Error-prone, slow, and easy to miss one.

## Why It Mattered

A homelab running identity-sensitive services a secrets manager, an SSO-capable reverse proxy, infrastructure dashboards with write access can't treat authentication as an afterthought. The constraint was the same as the rest of the fleet: self-hosted, no cloud identity provider, and operable by one person. The solution needed to support OIDC and SAML natively, enforce MFA at the platform level rather than per-app, and provide a single pane of glass for authentication events.
