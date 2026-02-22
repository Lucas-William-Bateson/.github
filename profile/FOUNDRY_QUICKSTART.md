# Foundry Deployment Quickstart

This guide explains how to quickly scaffold and deploy a Next.js App Router application to the Home Server via Foundry, both **with** and **without** WorkOS authentication.

---

## Part 1: Basic Deployment (Without Auth)

To deploy a standard Next.js application, you need to configure it to build as a Docker standalone image and provide the correct deployment descriptors for Foundry.

### 1. Configure Next.js for Standalone Output
In your `next.config.ts` (or `.js`), enable standalone output to significantly reduce the Docker image size:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
};

export default nextConfig;
```

### 2. Add the Multi-stage Dockerfile
Create a `docker/Dockerfile` in your repository:

```dockerfile
FROM node:20-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

RUN mkdir .next
RUN chown nextjs:nodejs .next

COPY --from=builder --chown=nextjs:nodejs /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000

CMD ["node", "server.js"]
```

### 3. Setup Docker Compose
Because `foundry-agent` relies on `docker-compose` to trigger the host-side `pass-cli` secrets watcher, always use a compose file even for single-container apps.

Create `docker/docker-compose.yml`:

```yaml
services:
  app:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    container_name: your_app_name
    ports:
      - "3000:3000" # Change the host port to your specific app port mapping
    restart: unless-stopped
    env_file:
      - ../secrets.env
    environment:
      - PORT=3000
      - HOSTNAME=0.0.0.0
```

### 4. Create the `foundry.toml`
Create `foundry.toml` in the root of your repository:

```toml
[build]
dockerfile = "docker/Dockerfile"
context = "."
timeout = 1800

[triggers]
branches = ["main"]
pull_requests = true

[deploy]
name = "your_app_name"
domains = ["yourapp.l3s.me"]
port = 3000
compose_file = "docker/docker-compose.yml"
env_file = "secrets.env"

[schedule]
cron = "0 0 4 * * * *"
branch = "main"
enabled = true
timezone = "UTC"

[env]
NODE_ENV = "production"
```

### 5. Secrets Management (Proton Pass)
Create a `secrets.env.template` in the root. The host-side `secrets-watcher.sh` will look for this file and run `pass-cli` to inject credentials into `secrets.env` before running docker-compose.

```env
# Regenerate: pass-cli inject secrets.env.template -o secrets.env
MY_SECRET_KEY={{ pass://Secrets management/your_app/my_secret_key }}
```

Add `secrets.env` to your `.gitignore`. Commit and push—Foundry will deploy it automatically!

---

## Part 2: Deploying with WorkOS AuthKit

When deploying a Next.js App Router behind a Cloudflare Tunnel reverse proxy, WorkOS AuthKit requires specific configurations to avoid `0.0.0.0` or `localhost` redirects.

### 1. Install AuthKit
```bash
npm install @workos-inc/authkit-nextjs
```

### 2. Lock the Redirect URI in Next.js Config
To guarantee that the redirect URL resolves correctly during the Docker build, hardcode it in your `next.config.ts`:

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

### 3. Add WorkOS Credentials to Secrets Template
In your `secrets.env.template`, reuse the global WorkOS credentials:

```env
WORKOS_CLIENT_ID={{ pass://Secrets management/WorkOS/client_id }}
WORKOS_API_KEY={{ pass://Secrets management/WorkOS/client_secret }}
WORKOS_COOKIE_PASSWORD={{ pass://Secrets management/WorkOS/cookie_secret }}
```

### 4. Configure the AuthKit Middleware
Create `src/middleware.ts`. This middleware explicitly grabs the `x-forwarded-host` header passed by the Cloudflare tunnel and overrides the internal Docker request URL. This forces AuthKit to generate correct, absolute callback URLs instead of internal `0.0.0.0` URLs.

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
    
    // Create a new request with the rewritten URL and Host header
    const newRequest = new NextRequest(url.toString(), request);
    newRequest.headers.set('host', forwardedHost);
    
    return authkitMiddleware({
      middlewareAuth: { enabled: true, unauthenticatedPaths: ['/'] },
      returnPathname: '/dashboard', // The page to redirect to after successful login
      redirectUri: `https://${forwardedHost}/api/auth/callback`,
    })(newRequest);
  }
  
  // Local development fallback
  return authkitMiddleware({
    middlewareAuth: { enabled: true, unauthenticatedPaths: ['/'] },
    returnPathname: '/dashboard',
    redirectUri: 'https://yourapp.l3s.me/api/auth/callback',
  })(request);
}

export const config = {
  matcher: ['/dashboard/:path*'], // Routes to protect
};
```

### 5. Configure the Auth Callback Route
Create the API route at `src/app/api/auth/[...workos]/route.ts`. The `baseURL` ensures that when WorkOS finishes authenticating the user, it redirects back to the public domain instead of the Docker container ID.

```typescript
import { handleAuth } from '@workos-inc/authkit-nextjs';

export const GET = handleAuth({
  baseURL: 'https://yourapp.l3s.me'
});
```

### 6. Implement Sign In (Landing Page)
In your unprotected `src/app/page.tsx`, generate the Sign In and Sign Up URLs. 

```tsx
import { getSignInUrl, getSignUpUrl } from '@workos-inc/authkit-nextjs';
import Link from 'next/link';

export default async function LandingPage() {
  let signInUrl = '#';
  let signUpUrl = '#';
  
  try {
    signInUrl = await getSignInUrl();
    signUpUrl = await getSignUpUrl();
  } catch (error) {
    // This catches missing environment variables (e.g., before Proton Pass injects them)
    return <div>Configuration Error: WorkOS credentials missing.</div>;
  }

  return (
    <main>
      <h1>Welcome to the App</h1>
      <Link href={signInUrl}>Sign In</Link>
      <Link href={signUpUrl}>Sign Up</Link>
    </main>
  );
}
```

### 7. Implement User Session (Dashboard)
In your protected `src/app/dashboard/page.tsx`, use `withAuth()` to retrieve the user's session and `signOut()` to log them out.

```tsx
import { signOut, withAuth } from '@workos-inc/authkit-nextjs';

export default async function DashboardPage() {
  const { user } = await withAuth();

  return (
    <main>
      <h1>Dashboard</h1>
      <p>Welcome back, {user?.firstName || user?.email}!</p>
      
      <form action={async () => {
        'use server';
        await signOut();
      }}>
        <button type="submit">Sign Out</button>
      </form>
    </main>
  );
}
```
