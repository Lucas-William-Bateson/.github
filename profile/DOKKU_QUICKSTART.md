# Dokku Deployment Quickstart

This guide explains how to deploy applications to the Home Server via Dokku, both **with** and **without** WorkOS authentication.

## Prerequisites

- SSH access to Dokku configured (see infrastructure docs)
- Git remote: `git remote add dokku dokku@dokku:<app-name>`

---

## Part 1: Basic Deployment (Dockerfile)

### 1. Create the Dokku App

```bash
ssh dokku apps:create your-app
ssh dokku domains:add your-app yourapp.l3s.me
ssh dokku ports:set your-app http:80:3000
```

### 2. Add a Dockerfile

Create `docker/Dockerfile` (or at repo root). For a Next.js standalone app:

```dockerfile
FROM node:20-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
RUN mkdir .next && chown nextjs:nodejs .next
COPY --from=builder --chown=nextjs:nodejs /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

### 3. Tell Dokku Where the Dockerfile Is

If your Dockerfile isn't at the repo root, set the builder config:

```bash
ssh dokku builder:set your-app selected dockerfile
ssh dokku builder-dockerfile:set your-app dockerfile-path docker/Dockerfile
```

### 4. Set Environment Variables

```bash
ssh dokku config:set your-app NODE_ENV=production PORT=3000 HOSTNAME=0.0.0.0
```

### 5. Deploy

```bash
git remote add dokku dokku@dokku:your-app
git push dokku main
```

### 6. Add Cloudflare Tunnel Route

Add a route for `yourapp.l3s.me → localhost:8080` in your Cloudflare tunnel configuration. Dokku's NGINX handles virtual host routing automatically.

---

## Part 2: Deploying with WorkOS AuthKit

### 1. Install AuthKit

```bash
npm install @workos-inc/authkit-nextjs
```

### 2. Lock the Redirect URI

In `next.config.ts`:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  env: {
    NEXT_PUBLIC_WORKOS_REDIRECT_URI: "https://yourapp.l3s.me/api/auth/callback",
  },
};

export default nextConfig;
```

### 3. Set WorkOS Credentials via Dokku Config

```bash
ssh dokku config:set your-app \
  WORKOS_CLIENT_ID=client_xxx \
  WORKOS_API_KEY=sk_xxx \
  WORKOS_COOKIE_PASSWORD=<32-char-secret>
```

### 4. Configure the AuthKit Middleware

Create `src/middleware.ts`:

```typescript
import { authkitMiddleware } from '@workos-inc/authkit-nextjs';
import { NextRequest } from 'next/server';

export default async function middleware(request: NextRequest) {
  const forwardedHost = request.headers.get('x-forwarded-host');
  
  if (forwardedHost && (request.url.includes('0.0.0.0') || request.url.includes('localhost') || request.url.includes('127.0.0.1'))) {
    const url = new URL(request.url);
    url.hostname = forwardedHost;
    url.port = '';
    url.protocol = request.headers.get('x-forwarded-proto') || 'https:';
    
    const newRequest = new NextRequest(url.toString(), request);
    newRequest.headers.set('host', forwardedHost);
    
    return authkitMiddleware({
      middlewareAuth: { enabled: true, unauthenticatedPaths: ['/'] },
      returnPathname: '/dashboard',
      redirectUri: `https://${forwardedHost}/api/auth/callback`,
    })(newRequest);
  }
  
  return authkitMiddleware({
    middlewareAuth: { enabled: true, unauthenticatedPaths: ['/'] },
    returnPathname: '/dashboard',
    redirectUri: 'https://yourapp.l3s.me/api/auth/callback',
  })(request);
}

export const config = {
  matcher: ['/dashboard/:path*'],
};
```

### 5. Configure the Auth Callback Route

Create `src/app/api/auth/[...workos]/route.ts`:

```typescript
import { handleAuth } from '@workos-inc/authkit-nextjs';

export const GET = handleAuth({
  baseURL: 'https://yourapp.l3s.me'
});
```

---

## Part 3: Adding a PostgreSQL Database

```bash
# Create database service
ssh dokku postgres:create your-db

# Link to app (auto-injects DATABASE_URL)
ssh dokku postgres:link your-db your-app
```

For Prisma, run migrations after deploy:

```bash
ssh dokku run your-app npx prisma migrate deploy
```

Or add a `Procfile` with a release phase:

```
release: npx prisma migrate deploy
web: node server.js
```

---

## Part 4: Deploying Pre-built Images

For apps like Uptime Kuma that use upstream images:

```bash
ssh dokku apps:create kuma
ssh dokku domains:add kuma status.l3s.me
ssh dokku ports:set kuma http:80:3001
ssh dokku storage:mount kuma /var/lib/dokku/data/storage/kuma-data:/app/data
ssh dokku git:from-image kuma louislam/uptime-kuma:1
```

---

## Common Dokku Commands

```bash
# App management
ssh dokku apps:list
ssh dokku ps:report <app>
ssh dokku logs <app> --tail

# Configuration
ssh dokku config:show <app>
ssh dokku config:set <app> KEY=value
ssh dokku config:unset <app> KEY

# Domains & ports
ssh dokku domains:report <app>
ssh dokku domains:add <app> domain.l3s.me
ssh dokku ports:set <app> http:80:3000

# Database
ssh dokku postgres:info <db>
ssh dokku postgres:connect <db>

# Rebuild & restart
ssh dokku ps:rebuild <app>
ssh dokku ps:restart <app>
```

This is useful for monorepo setups or when you only want CI to run on PRs headed for production branches.
