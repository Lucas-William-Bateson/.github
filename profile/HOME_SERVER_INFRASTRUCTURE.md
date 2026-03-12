# Home Server Infrastructure Documentation

> **Last Updated:** March 2026
> **Owner:** Lucas William Bateson  
> **Base Domain:** `l3s.me`

---

## 📍 Hardware Overview

| Component         | Specification                     |
| ----------------- | --------------------------------- |
| **Device**        | Mac Mini (Mac16,10)               |
| **Model Number**  | MU9D3DH/A                         |
| **Chip**          | Apple M4                          |
| **CPU Cores**     | 10 (4 performance + 6 efficiency) |
| **Memory**        | 16 GB                             |
| **OS**            | macOS                             |
| **Location**      | Home network                      |
| **Remote Access** | Tailscale (100.94.15.7)           |

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
│  • l3s.me              → localhost:8080 (Dokku NGINX)                       │
│  • portfolio.l3s.me    → localhost:8080                                     │
│  • lucasbateson.com    → localhost:8080                                     │
│  • notes.l3s.me        → localhost:8080                                     │
│  • budget.l3s.me       → localhost:8080                                     │
│  • status.l3s.me       → localhost:8080                                     │
│  • mini.l3s.me         → ssh://localhost:22                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MAC MINI (M4)                                      │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     COLIMA VM (Ubuntu 24.04)                          │  │
│  │              6 CPUs · 12 GB RAM · 80 GB Disk · aarch64               │  │
│  │              macOS Virtualization.Framework                           │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     DOCKER ENGINE                               │  │  │
│  │  │                                                                 │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐   │  │  │
│  │  │  │              DOKKU (0.37.7)                             │   │  │  │
│  │  │  │         Privileged container + docker.sock              │   │  │  │
│  │  │  │         Ports: 3022→22, 8080→80, 8443→443              │   │  │  │
│  │  │  │         Volume: dokku-data → /mnt/dokku                │   │  │  │
│  │  │  │                                                         │   │  │  │
│  │  │  │    ┌───────────┐ ┌───────────┐ ┌───────────┐          │   │  │  │
│  │  │  │    │ Portfolio  │ │   Blog    │ │  Budget   │          │   │  │  │
│  │  │  │    │  :3000    │ │   :80     │ │  :3002    │          │   │  │  │
│  │  │  │    └───────────┘ └───────────┘ └───────────┘          │   │  │  │
│  │  │  │                                                         │   │  │  │
│  │  │  │    ┌───────────┐ ┌───────────┐                        │   │  │  │
│  │  │  │    │   Kuma    │ │ Postgres  │                        │   │  │  │
│  │  │  │    │  :3001    │ │  :5432    │                        │   │  │  │
│  │  │  │    └───────────┘ └───────────┘                        │   │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘   │  │  │
│  │  │                                                                 │  │  │
│  │  │  ┌───────────┐                                                 │  │  │
│  │  │  │cloudflared│                                                 │  │  │
│  │  │  └───────────┘                                                 │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Authentication Architecture

Authentication is handled on a per-app basis using **WorkOS AuthKit**.

```text
  User Request
       │
       ▼
┌──────────────┐     ┌─────────────────────────────────────────────────┐
│   Cloudflare │────▶│             Dokku NGINX                         │
│    Tunnel    │     │         (virtual host routing)                  │
└──────────────┘     └──────────────┬──────────────────────────────────┘
                                    │
                                    ▼
                     ┌─────────────────────────────────────────────────┐
                     │             Protected Service                   │
                     │           (e.g., Budget app)                    │
                     │                                                 │
                     │  ┌──────────────┐       ┌─────────────────┐     │
                     │  │   App Auth   │──────▶│ WorkOS Identity │     │
                     │  │ (Middleware) │       │   Provider      │     │
                     │  └──────────────┘       └─────────────────┘     │
                     └─────────────────────────────────────────────────┘
```

| Component           | Purpose                                                 |
| ------------------- | ------------------------------------------------------- |
| **WorkOS**          | Identity Provider / SSO for per-app authentication      |
| **Session Cookies** | Secure, HTTP-only HS256 JWT cookies verified by the app |

---

## 🔑 Secrets Management

Environment variables are set per-app via Dokku's config system:

```bash
ssh dokku config:set <app> KEY=value
```

Sensitive values (API keys, database credentials) are stored in Dokku's config and injected at runtime. Database connection strings are managed automatically by the dokku-postgres plugin via linked services.

---

## 📦 Services Inventory

### 1. Portfolio Website

| Property          | Value                                                                          |
| ----------------- | ------------------------------------------------------------------------------ |
| **Dokku App**     | `portfolio`                                                                    |
| **Domains**       | `l3s.me`, `portfolio.l3s.me`, `lucasbateson.com`, `portfolio.lucasbateson.com` |
| **Internal Port** | 3000                                                                           |
| **Repository**    | `Lucas-William-Bateson/portfolio`                                              |
| **Framework**     | Astro 4.x + Tailwind                                                           |
| **Builder**       | Dockerfile (`docker/Dockerfile`)                                               |
| **Runtime**       | `serve` (static site)                                                          |

---

### 2. Blog / Notes

| Property          | Value                            |
| ----------------- | -------------------------------- |
| **Dokku App**     | `blog`                           |
| **Domain**        | `notes.l3s.me`                   |
| **Internal Port** | 80                               |
| **Repository**    | `Lucas-William-Bateson/blog`     |
| **Framework**     | Astro 5.x + Tailwind + pnpm      |
| **Builder**       | Dockerfile (`docker/Dockerfile`) |
| **Runtime**       | nginx (static site)              |

---

### 3. Budget

| Property          | Value                                                |
| ----------------- | ---------------------------------------------------- |
| **Dokku App**     | `budget`                                             |
| **Domain**        | `budget.l3s.me`                                      |
| **Internal Port** | 3002                                                 |
| **Repository**    | `Lucas-William-Bateson/budget`                       |
| **Framework**     | Next.js 16 + Prisma 7 + React 19 + Designsystemet    |
| **Builder**       | Dockerfile (`docker/Dockerfile`)                     |
| **Database**      | PostgreSQL 16 (dokku-postgres, service: `budget-db`) |
| **Auth**          | WorkOS AuthKit                                       |

---

### 4. Uptime Kuma (Status Monitoring)

| Property          | Value                                                    |
| ----------------- | -------------------------------------------------------- |
| **Dokku App**     | `kuma`                                                   |
| **Domain**        | `status.l3s.me`                                          |
| **Internal Port** | 3001                                                     |
| **Image**         | `louislam/uptime-kuma:1` (deployed via `git:from-image`) |
| **Storage**       | `/var/lib/dokku/data/storage/kuma-data:/app/data`        |

---

## 🗄️ Database Overview

| Database    | Service | Plugin         | Image                |
| ----------- | ------- | -------------- | -------------------- |
| `budget-db` | Budget  | dokku-postgres | `postgres:16-alpine` |

Managed by the [dokku-postgres](https://github.com/dokku/dokku-postgres) plugin. Connection string is auto-injected as `DATABASE_URL` when linked to an app.

---

## 🔄 Deployment Pipeline

### Git Push Deployment

All Dockerfile-based apps are deployed via `git push`:

```text
Developer Machine → git push dokku main → Dokku builds Dockerfile → Deploy + NGINX routing
```

### Pre-built Image Deployment

Apps using pre-built images (e.g., Uptime Kuma) are deployed via:

```bash
ssh dokku git:from-image <app> <image>:<tag>
```

### Dokku SSH Access

From the dev machine, Dokku is accessed via SSH (configured in `~/.ssh/config`):

```
Host dokku
  HostName 100.94.15.7
  Port 3022
  User dokku
  IdentityFile ~/.ssh/id_ed25519
  RequestTTY no
```

Git remotes for each app:

```bash
git remote add dokku dokku@dokku:<app-name>
```

---

## 🌍 Domain Configuration

### Primary Domain: `l3s.me`

| Subdomain          | Dokku App |
| ------------------ | --------- |
| `l3s.me`           | portfolio |
| `portfolio.l3s.me` | portfolio |
| `notes.l3s.me`     | blog      |
| `budget.l3s.me`    | budget    |
| `status.l3s.me`    | kuma      |

### Secondary Domain: `lucasbateson.com`

| Subdomain                    | Dokku App |
| ---------------------------- | --------- |
| `lucasbateson.com`           | portfolio |
| `portfolio.lucasbateson.com` | portfolio |

All domains route through the Cloudflare Tunnel to Dokku's NGINX on port 8080, which routes to the correct app container based on the `Host` header.

### SSH Access

| Subdomain     | Target                                      |
| ------------- | ------------------------------------------- |
| `mini.l3s.me` | `ssh://localhost:22` (Cloudflare SSH proxy) |

---

## 🐳 Docker-in-Docker Volume Fix

Dokku runs as a privileged container with Docker socket access, meaning its plugins (like dokku-postgres) create **sibling containers** on the host Docker engine. Volume paths referenced from inside the Dokku container don't map correctly to host paths.

**Fix:** Set `DOKKU_LIB_HOST_ROOT` so the postgres plugin generates correct host-side volume mount paths:

```bash
# Inside the Dokku container:
DOKKU_LIB_HOST_ROOT=/var/lib/docker/volumes/dokku-data/_data/var/lib/dokku
```

This is set in `/etc/default/dokku`, `/home/dokku/.bashrc`, `/home/dokku/.profile`, and `/etc/environment` inside the Dokku container.

---

## 🔧 Maintenance Tasks

### Common Commands

```bash
# SSH to Mac Mini (via Tailscale)
ssh lucas@100.94.15.7

# List all Dokku apps
ssh dokku apps:list

# View app status
ssh dokku ps:report <app>

# View app logs
ssh dokku logs <app> --tail

# Restart an app
ssh dokku ps:restart <app>

# Set environment variables
ssh dokku config:set <app> KEY=value

# View app domains
ssh dokku domains:report <app>

# View all containers on the server
ssh lucas@100.94.15.7 docker ps -a

# Enter the Dokku container
ssh lucas@100.94.15.7 docker exec -it dokku bash

# Postgres management
ssh dokku postgres:info budget-db
ssh dokku postgres:connect budget-db
```

### Deploy an App

```bash
# Dockerfile-based apps
cd ~/Github/<repo>
git remote add dokku dokku@dokku:<app-name>  # one-time setup
git push dokku main

# Pre-built image apps
ssh dokku git:from-image <app> <image>:<tag>
```

### Backup Locations

| Data                 | Location                                                         |
| -------------------- | ---------------------------------------------------------------- |
| Dokku app data       | `dokku-data` Docker volume                                       |
| Kuma monitoring data | `/var/lib/dokku/data/storage/kuma-data` (inside Dokku container) |
| Budget PostgreSQL    | Managed by dokku-postgres (`budget-db` service)                  |

---
