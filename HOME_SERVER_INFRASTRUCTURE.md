# Home Server Infrastructure Documentation

> **Last Updated:** January 27, 2026  
> **Owner:** Lucas William Bateson  
> **Base Domain:** `l3s.me`

---

## 📍 Hardware Overview

| Component | Specification |
|-----------|---------------|
| **Device** | Mac Mini (Mac16,10) |
| **Model Number** | MU9D3DH/A |
| **Chip** | Apple M4 |
| **CPU Cores** | 10 (4 performance + 6 efficiency) |
| **Memory** | 16 GB |
| **OS** | macOS |
| **Location** | Home network |
| **Remote Access** | Tailscale (100.94.15.7) |

---

## 🌐 Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLOUDFLARE TUNNEL                                    │
│                    (cloudflared container)                                   │
│                                                                              │
│  Domains:                                                                    │
│  • foundry.l3s.me      → localhost:8081                                     │
│  • auth.l3s.me         → localhost:8080                                     │
│  • gate.l3s.me         → localhost:4180                                     │
│  • n8n.l3s.me          → localhost:5678                                     │
│  • status.l3s.me       → localhost:3001                                     │
│  • notes.l3s.me        → localhost:80                                       │
│  • l3s.me              → localhost:3000                                     │
│  • portfolio.l3s.me    → localhost:3000                                     │
│  • lucasbateson.com    → localhost:3000                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MAC MINI (M4)                                      │
│                         Docker Desktop                                       │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      DOCKER CONTAINERS                               │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │   Foundry    │  │   Keycloak   │  │ Auth Gateway │              │   │
│  │  │   :8081      │  │    :8080     │  │    :4180     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │     n8n      │  │ Uptime Kuma  │  │  Portfolio   │              │   │
│  │  │    :5678     │  │    :3001     │  │    :3000     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐                                │   │
│  │  │     Blog     │  │   Registry   │                                │   │
│  │  │     :80      │  │    :5001     │                                │   │
│  │  └──────────────┘  └──────────────┘                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Authentication Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AUTHENTICATION FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────┘

  User Request
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Cloudflare │────▶│ Auth Gateway │────▶│   Keycloak   │
│    Tunnel    │     │ (OAuth2Proxy)│     │    (OIDC)    │
│              │     │  gate.l3s.me │     │ auth.l3s.me  │
└──────────────┘     └──────────────┘     └──────────────┘
                            │
                            │ Authenticated
                            ▼
                     ┌──────────────┐
                     │  Protected   │
                     │   Services   │
                     │ (n8n, etc.)  │
                     └──────────────┘
```

### Authentication Components

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| **Keycloak** | Identity Provider (OIDC) | Realm: `personal-org` |
| **OAuth2-Proxy** | Authentication gateway | Client: `auth-gateway` |
| **Cookie Domain** | SSO across subdomains | `.l3s.me` |

---

## 📦 Services Inventory

### 1. Foundry (CI/CD Platform)

| Property | Value |
|----------|-------|
| **Domain** | `foundry.l3s.me` |
| **Port** | 8081 |
| **Repository** | `Lucas-William-Bateson/foundry` |
| **Image** | Custom (Rust + React) |
| **Database** | PostgreSQL 16 (foundry_postgres_data) |

**Components:**
- `foundryd` - Main server (API + Web UI)
- `foundry-agent` - Build agent (claims and executes jobs)
- `postgres` - Job and repo metadata storage
- `cloudflared` - Cloudflare tunnel daemon

**Functionality:**
- GitHub webhook receiver
- Automatic deployments on push
- Build job queuing and execution
- Docker compose deployments
- Scheduled builds (cron)
- Real-time build logs

**Connections:**
- GitHub App (webhook + clone authentication)
- Docker socket (for running container deployments)
- Cloudflare API (tunnel management)

---

### 2. Keycloak (Identity Server)

| Property | Value |
|----------|-------|
| **Domain** | `auth.l3s.me` |
| **Port** | 8080 |
| **Repository** | `Lucas-William-Bateson/identity-keycloak` |
| **Image** | Custom (based on Keycloak official) |
| **Database** | PostgreSQL 16 (keycloak-db) |

**Configuration:**
- Realm: `personal-org`
- Admin Console: `https://auth.l3s.me/admin`
- Proxy Headers: `xforwarded`
- Strict Hostname: Disabled

**Clients Configured:**
- `auth-gateway` - OAuth2-Proxy client for SSO

**Connections:**
- Auth Gateway (OIDC provider)
- Protected applications (token validation)

---

### 3. Auth Gateway (OAuth2-Proxy)

| Property | Value |
|----------|-------|
| **Domain** | `gate.l3s.me` |
| **Port** | 4180 |
| **Repository** | `Lucas-William-Bateson/auth-gateway` |
| **Image** | Custom (OAuth2-Proxy based) |
| **Env File** | `/Users/lucas/foundry/auth-gateway.env` |

**Configuration:**
```
Provider: OIDC
Issuer: https://auth.l3s.me/realms/personal-org
Cookie Domain: .l3s.me
Redirect URL: https://gate.l3s.me/oauth2/callback
Upstream: static://202
```

**Features:**
- SSO across all `.l3s.me` subdomains
- X-Auth headers injection
- Access token passthrough
- Skip auth for health endpoints

**Connections:**
- Keycloak (authentication)
- Protected services (via nginx auth_request)

---

### 4. n8n (Workflow Automation)

| Property | Value |
|----------|-------|
| **Domain** | `n8n.l3s.me` |
| **Port** | 5678 |
| **Repository** | `Lucas-William-Bateson/n8n` |
| **Image** | `docker.n8n.io/n8nio/n8n` |
| **Data Volume** | `n8n_data` |

**Configuration:**
- Webhook URL: `https://n8n.l3s.me/`
- Runners Enabled: Yes
- Resource Limits: 1 CPU, 1GB Memory

**Scheduled Rebuild:** Daily at 02:00 UTC

**Use Cases:**
- Automation workflows
- Webhook integrations
- Data processing pipelines

---

### 5. Uptime Kuma (Status Monitoring)

| Property | Value |
|----------|-------|
| **Domain** | `status.l3s.me` |
| **Port** | 3001 |
| **Repository** | `Lucas-William-Bateson/uptime-kuma` (fork) |
| **Image** | `louislam/uptime-kuma:1` |
| **Data Volume** | `uptime-kuma` |

**Scheduled Rebuild:** Weekly on Monday at 01:00 UTC

**Functionality:**
- HTTP(S) endpoint monitoring
- TCP port monitoring
- DNS monitoring
- Status page generation
- Alert notifications

---

### 6. Portfolio Website

| Property | Value |
|----------|-------|
| **Domains** | `l3s.me`, `portfolio.l3s.me`, `lucasbateson.com`, `portfolio.lucasbateson.com` |
| **Port** | 3000 |
| **Repository** | `Lucas-William-Bateson/portfolio` |
| **Image** | Custom (Astro + Tailwind) |
| **Framework** | Astro |

**Scheduled Rebuild:** Daily at 05:00 UTC

**Features:**
- Personal portfolio/CV
- Static site generation
- Multiple domain support

---

### 7. Blog / Notes

| Property | Value |
|----------|-------|
| **Domain** | `notes.l3s.me` |
| **Port** | 80 |
| **Repository** | `Lucas-William-Bateson/blog` |
| **Image** | Custom (Astro) |
| **Framework** | Astro |

**Scheduled Rebuild:** Daily at 03:00 UTC

**Features:**
- Personal blog/notes
- Content collection
- RSS feed

---

### 8. Docker Registry

| Property | Value |
|----------|-------|
| **Port** | 5001 (internal) |
| **Image** | `registry:2` |
| **Data Volume** | `registry-data` |

**Functionality:**
- Local container image storage
- Push/pull for custom images

---

## 🗄️ Database Overview

| Database | Service | Port | Volume |
|----------|---------|------|--------|
| `foundry` | Foundry Server | 5432 | `foundry_postgres_data` |
| `keycloak` | Keycloak | 5432 | `keycloak-db` |

Both databases run PostgreSQL 16 Alpine.

---

## 🔄 Deployment Pipeline

### Automatic Deployments

All services are deployed automatically via Foundry when pushing to their respective repositories:

```
GitHub Push → Webhook → Foundry Server → Agent Claims Job → Docker Build → Deploy
```

### Foundry Configuration (foundry.toml)

Each repository contains a `foundry.toml` that defines:
- Build configuration (Dockerfile, context, timeout)
- Trigger rules (branches, pull requests)
- Deploy settings (name, domain, port, compose file)
- Schedule (cron expression for periodic rebuilds)
- Environment variables

### Env Files Location

Sensitive environment variables are stored on the host at `/Users/lucas/foundry/`:

| File | Service |
|------|---------|
| `secrets.env` | Foundry (GitHub App secrets, tunnel token) |
| `auth-gateway.env` | OAuth2-Proxy (client secret, cookie secret) |
| `identity-keycloak.env` | Keycloak (if needed) |

---

## 🌍 Domain Configuration

### Primary Domain: `l3s.me`

| Subdomain | Service | Port |
|-----------|---------|------|
| `l3s.me` | Portfolio | 3000 |
| `portfolio.l3s.me` | Portfolio | 3000 |
| `foundry.l3s.me` | Foundry | 8081 |
| `auth.l3s.me` | Keycloak | 8080 |
| `gate.l3s.me` | Auth Gateway | 4180 |
| `n8n.l3s.me` | n8n | 5678 |
| `status.l3s.me` | Uptime Kuma | 3001 |
| `notes.l3s.me` | Blog | 80 |

### Secondary Domain: `lucasbateson.com`

| Subdomain | Service | Port |
|-----------|---------|------|
| `lucasbateson.com` | Portfolio | 3000 |
| `portfolio.lucasbateson.com` | Portfolio | 3000 |

All domains route through the Cloudflare Tunnel to the Mac Mini.

---

## 🐳 Docker Networks

| Network | Purpose | Services |
|---------|---------|----------|
| `foundry_default` | Foundry stack | foundryd, agent, postgres, cloudflared |
| `keycloak-net` | Identity stack | keycloak, postgres-keycloak |
| `auth-net` | Auth gateway | oauth2-proxy |
| `n8n-network` | Workflow automation | n8n |

---

## 📊 Scheduled Builds

| Service | Schedule | Timezone | Branch |
|---------|----------|----------|--------|
| Blog | Daily 03:00 | UTC | main |
| n8n | Daily 02:00 | UTC | main |
| Portfolio | Daily 05:00 | UTC | main |
| Uptime Kuma | Weekly Mon 01:00 | UTC | master |

---

## 🔧 Maintenance Tasks

### Common Commands

```bash
# SSH to Mac Mini (via Tailscale)
ssh lucas@100.94.15.7

# Set Docker path
export PATH=/Applications/Docker.app/Contents/Resources/bin:$PATH

# View all containers
docker ps -a

# View logs for a service
docker logs <container-name>

# Restart a service
docker restart <container-name>

# View Foundry jobs
docker exec foundry-postgres-1 psql -U foundry -d foundry -c "SELECT id, status, created_at FROM job ORDER BY created_at DESC LIMIT 10;"
```

### Backup Locations

| Data | Location |
|------|----------|
| Foundry DB | `foundry_postgres_data` volume |
| Keycloak DB | `keycloak-db` volume |
| n8n Workflows | `n8n_data` volume |
| Uptime Kuma | `uptime-kuma` volume |
| Docker Registry | `registry-data` volume |

---

## 🚨 Troubleshooting

### Service Won't Start
1. Check logs: `docker logs <container>`
2. Check health: `docker ps` (look at STATUS)
3. Verify dependencies are healthy

### Cloudflare Tunnel Issues
1. Check `foundry-cloudflared-1` container
2. Verify `TUNNEL_TOKEN` in `secrets.env`
3. Check Cloudflare dashboard for tunnel status

### Authentication Problems
1. Verify Keycloak is running and healthy
2. Check OAuth2-Proxy logs
3. Confirm cookie domain matches (`.l3s.me`)
4. Verify client secret matches in Keycloak

### Foundry Deployment Failures
1. Check build logs in Foundry UI
2. Verify env file exists at `/Users/lucas/foundry/<app>.env`
3. Check agent has access to Docker socket
4. Verify compose file syntax

---

## 📝 Version History

| Date | Change |
|------|--------|
| Jan 27, 2026 | Added Auth Gateway (OAuth2-Proxy) |
| Jan 27, 2026 | Added Keycloak identity server |
| Jan 2026 | Initial Foundry setup with all services |

---

## 📚 Related Documentation

- [Foundry Setup & Migration Guide](./SETUP_AND_MIGRATION.md)
- [OAuth2-Proxy Configuration](https://oauth2-proxy.github.io/oauth2-proxy/)
- [Keycloak Admin Guide](https://www.keycloak.org/documentation)
- [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
