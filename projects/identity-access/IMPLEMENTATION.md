# IMPLEMENTATION: Zero-Trust Identity Platform (Authentik SSO)

## 1. Architecture

Authentik runs on Home One (VM 200) on Node-C alongside its two dependencies, PostgreSQL and Redis. Co-locating all three on the same host prevents split-brain identity failures where the IdP is up but its database isn't reachable.

```
User Login → Authentik (192.168.20.10:9000) → OIDC/SAML → Application
                │
                ├── MFA Challenge (TOTP/WebAuthn)
                ├── Policy Evaluation (group, IP, device)
                └── Audit Log → PostgreSQL
```

The stack runs as a Docker Compose deployment:

| Container | Image | Role |
|-----------|-------|------|
| authentik-server | ghcr.io/goauthentik/server:latest | Web UI, API, provider endpoints |
| authentik-worker | ghcr.io/goauthentik/server:latest | Background tasks, outpost management |
| postgresql | postgres:15 | User directory, flow config, audit logs |
| redis | redis:alpine | Session storage, caching |
| portainer | portainer/portainer-ce:latest | Container management UI |

## 2. Deployment

Generate the secret key for Authentik's internal encryption:

```bash
openssl rand -hex 32  # Save to .env as AUTHENTIK_SECRET_KEY
```

The Docker Compose environment variables point both the server and worker containers at the shared PostgreSQL and Redis instances. After `docker compose up -d`, initial setup is at:

```
https://192.168.20.10:9443/if/flow/initial-setup/
```

The setup wizard creates the `akadmin` account. After that, create a real admin user and disable the default.

## 3. OIDC/SAML Provider Setup

Each integrated service gets its own provider in Authentik. The general pattern:

1. **Authentik Admin → Applications → Providers → Create**
2. Select provider type: OAuth2/OpenID (most services) or SAML (Wazuh Dashboard)
3. Set client type to **Confidential**
4. Register the service's redirect URI
5. Copy the generated client ID and client secret into the service's auth config

**OIDC provider**: used for Grafana, n8n, Vaultwarden, OpenWebUI, Proxmox, Portainer. Each provider exposes standard endpoints:

```
Authorization: http://192.168.20.10:9000/application/o/authorize/
Token:         http://192.168.20.10:9000/application/o/token/
User Info:     http://192.168.20.10:9000/application/o/userinfo/
```

**SAML provider**: used for the Wazuh Dashboard. Authentik provides the IdP metadata URL; the dashboard is configured with the SP entity ID. Users authenticate via Authentik and land in Wazuh with role mappings applied via Authentik group membership.

**Forward auth (catch-all)**: services that don't natively support SSO sit behind NPM with Authentik's outpost. The outpost intercepts unauthenticated requests before they reach the application:

```nginx
# NPM Advanced Config per proxy host
location /outpost.goauthentik.io {
    proxy_pass http://192.168.20.10:9000/outpost.goauthentik.io;
    proxy_set_header Host $host;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
}
auth_request /outpost.goauthentik.io/auth/nginx;
```

## 4. Service Integrations

### Grafana: OIDC

In Authentik, create an OAuth2/OpenID provider with redirect URI `http://192.168.20.40:3000/login/generic_oauth`. In Grafana's `grafana.ini`:

```ini
[server]
domain = 192.168.20.40
root_url = http://192.168.20.40:3000/

[auth.generic_oauth]
enabled = true
name = Authentik
client_id = YOUR_CLIENT_ID
client_secret = YOUR_CLIENT_SECRET
scopes = openid profile email
auth_url = http://192.168.20.10:9000/application/o/authorize/
token_url = http://192.168.20.10:9000/application/o/token/
api_url = http://192.168.20.10:9000/application/o/userinfo/
```

### Portainer: OAuth2

In Portainer → Settings → Authentication → OAuth:

```
Authorization URL: http://192.168.20.10:9000/application/o/authorize/
Access Token URL:  http://192.168.20.10:9000/application/o/token/
Resource URL:      http://192.168.20.10:9000/application/o/userinfo/
```

Redirect URIs in Authentik must include both the IP and the domain: `https://192.168.20.10:9443/` and `https://portainer.tima.dev/`.

## 5. MFA Configuration

MFA is enforced at the **Authentik flow level**, not at individual applications. This is the key architectural decision add the MFA stage to the default authentication flow and every application inherits it automatically. No per-app configuration needed.

### Flow Configuration

```
Authentik → Flows → default-authentication-flow → Edit Stages:

1. Identification Stage (username/email)
2. Password Stage
3. MFA TOTP Stage ← Added here
```

### Group Structure

| Group | MFA | Access |
|-------|-----|--------|
| `homelab-admins` | TOTP + optional WebAuthn (YubiKey) | All management surfaces |
| `homelab-users` | TOTP | SSO with standard access |
| `service-accounts` | Token-based | API access, no interactive login |

Results after deployment: credentials per service dropped from 12+ to 1, MFA coverage went from ~30% to 100% enforced, and every authentication event is now logged in PostgreSQL.

## 6. The Grafana OAuth Fix

**Symptom:** Clicking "Sign in with Authentik" on Grafana returned `redirect_uri_mismatch` from Authentik. The registered redirect URI and the OAuth config both showed `http://192.168.20.40:3000/login/generic_oauth`. Everything appeared to match.

**Diagnosis:** Browser dev tools → Network tab → inspect the OAuth redirect request. The `redirect_uri` parameter Grafana was actually sending: `http://localhost:3000/login/generic_oauth`. Grafana was sending `localhost` instead of its real IP.

**Root cause:** Grafana constructs the OAuth `redirect_uri` dynamically from its `[server]` section's `root_url` setting. When `root_url` is not explicitly set or is commented out with a semicolon, which it is by default in a fresh install Grafana falls back to `localhost`. This is correct from Grafana's own perspective but completely wrong for any external OAuth callback.

**The trap:** The `root_url` setting is in the `[server]` section, not the `[auth.generic_oauth]` section. When you're debugging OAuth, you're looking at the auth block. The server section feels unrelated. And in a fresh `grafana.ini`, the line is commented out with a semicolon it looks intentional, like Grafana is using sensible defaults. It isn't.

**Fix:** Uncomment and set `root_url` explicitly:

```ini
[server]
domain = 192.168.20.40
root_url = http://192.168.20.40:3000/
```

Restart Grafana, clear browser cache, and the OAuth redirect completes successfully. MFA challenge appears (enforced at the Authentik flow level), and Grafana receives the user info and creates the account.
