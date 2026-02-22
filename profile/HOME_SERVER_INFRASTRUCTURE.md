# Home Server Infrastructure Documentation

> **Last Updated:** February 2026  
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

```text
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
│  │  │   Foundry    │  │     n8n      │  │ Uptime Kuma  │              │   │
│  │  │   :8081      │  │    :5678     │  │    :3001     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  Portfolio   │  │     Blog     │  │   Registry   │              │   │
│  │  │    :3000     │  │     :80      │  │    :5001     │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Authentication Architecture

Authentication is handled on a per-app basis using **WorkOS**, replacing the legacy centralized Keycloak and OAuth2-Proxy gateway setup.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AUTHENTICATION FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────┘

  User Request
       │
       ▼
┌──────────────┐     ┌─────────────────────────────────────────────────┐
│   Cloudflare │────▶│             Protected Service                   │
│    Tunnel    │     │          (e.g., Foundry, n8n)                   │
└──────────────┘     │                                                 │
                     │  ┌──────────────┐       ┌─────────────────┐     │
                     │  │   App Auth   │──────▶│ WorkOS Identity │     │
                     │  │ (Middleware) │       │   Provider      │     │
                     │  └──────────────┘       └─────────────────┘     │
                     └─────────────────────────────────────────────────┘
```

### Authentication Components

| Component | Purpose | Details |
|-----------|---------|---------|
| **WorkOS** | Identity Provider / SSO | Replaced Keycloak for all per-app authentication needs. |
| **Session Cookies** | Session Management | Secure, HTTP-only HS256 JWT cookies verified by the app. |

---

## 🔑 Secrets Management

Secrets are injected and managed securely using **Proton Pass** (via `pass-cli`) and local environment templates, ensuring sensitive credentials are never hardcoded.

- **Host-side Injection**: Services like Foundry use a launchd watcher and `pass-cli` to inject credentials from Proton Pass dynamically.
- **Environment Templates**: Repositories (such as `blog`, `n8n`, and `portfolio`) utilize `secrets.env` configurations and templates for robust secret provisioning.

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
- **Auth**: WorkOS integration with HS256 session cookies
- **Secrets**: Proton Pass host-side secret injection via watcher

**Connections:**
- GitHub App (webhook + clone authentication)
- Docker socket (for running container deployments)
- Cloudflare API (tunnel management)

---

### 2. n8n (Workflow Automation)

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
- Secrets: Managed via `secrets.env` configuration

**Scheduled Rebuild:** Daily at 02:00 UTC

---

### 3. Uptime Kuma (Status Monitoring)

| Property | Value |
|----------|-------|
| **Domain** | `status.l3s.me` |
| **Port** | 3001 |
| **Repository** | `Lucas-William-Bateson/uptime-kuma` (fork) |
| **Image** | `louislam/uptime-kuma:1` |
| **Data Volume** | `uptime-kuma` |

**Scheduled Rebuild:** Weekly on Monday at 01:00 UTC

---

### 4. Portfolio Website

| Property | Value |
|----------|-------|
| **Domains** | `l3s.me`, `portfolio.l3s.me`, `lucasbateson.com`, `portfolio.lucasbateson.com` |
| **Port** | 3000 |
| **Repository** | `Lucas-William-Bateson/portfolio` |
| **Image** | Custom (Astro + Tailwind) |
| **Framework** | Astro |

**Scheduled Rebuild:** Daily at 05:00 UTC
**Secrets:** Managed via `secrets.env` configuration

---

### 5. Blog / Notes

| Property | Value |
|----------|-------|
| **Domain** | `notes.l3s.me` |
| **Port** | 80 |
| **Repository** | `Lucas-William-Bateson/blog` |
| **Image** | Custom (Astro) |
| **Framework** | Astro |

**Scheduled Rebuild:** Daily at 03:00 UTC
**Secrets:** Managed via `secrets.env` configuration

---

### 6. Docker Registry

| Property | Value |
|----------|-------|
| **Port** | 5001 (internal) |
| **Image** | `registry:2` |
| **Data Volume** | `registry-data` |

---

## 🗄️ Database Overview

| Database | Service | Port | Volume |
|----------|---------|------|--------|
| `foundry` | Foundry Server | 5432 | `foundry_postgres_data` |

Database runs PostgreSQL 16 Alpine.

---

## 🔄 Deployment Pipeline

### Automatic Deployments

All services are deployed automatically via Foundry when pushing to their respective repositories:

```text
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

Sensitive environment variables are securely stored or dynamically injected using templates, `secrets.env`, and `pass-cli`. Host-side configurations typically reside in `/Users/lucas/foundry/`.

---

## 🌍 Domain Configuration

### Primary Domain: `l3s.me`

| Subdomain | Service | Port |
|-----------|---------|------|
| `l3s.me` | Portfolio | 3000 |
| `portfolio.l3s.me` | Portfolio | 3000 |
| `foundry.l3s.me` | Foundry | 8081 |
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
| n8n Workflows | `n8n_data` volume |
| Uptime Kuma | `uptime-kuma` volume |
| Docker Registry | `registry-data` volume |

---