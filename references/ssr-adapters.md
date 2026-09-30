# SSR & Adapters

Astro supports on-demand server rendering with various deployment adapters.

## Contents
- Output Modes
- Opting In/Out of Prerendering
- Available Adapters
- SSR Features
- Server Islands
- Sessions
- API Endpoints
- Context and Locals
- Best Practices

## Output Modes

### Static (Default)

```javascript
// astro.config.mjs
export default defineConfig({
  output: 'static', // Default - all pages prerendered
});
```

All pages built at build time. No server required.

### Server (Full SSR)

```javascript
// astro.config.mjs
import node from '@astrojs/node';

export default defineConfig({
  output: 'server', // All pages rendered on-demand
  adapter: node({
    mode: 'standalone',
  }),
});
```

All pages rendered per-request. Requires an adapter.

### Mixed static + on-demand (the old "hybrid")

> `output: 'hybrid'` was **removed in Astro 5**. Keep the default `output: 'static'`, add an adapter, and opt individual routes into on-demand rendering with `export const prerender = false`.

```javascript
// astro.config.mjs
import vercel from '@astrojs/vercel';

export default defineConfig({
  // output: 'static' is the default — no need to set it
  adapter: vercel(),
});
```

## Opting In/Out of Prerendering

### In Static Mode (opt a route into on-demand rendering)

```typescript
// src/pages/api/time.ts
export const prerender = false; // Server-rendered

export const GET = () => {
  return new Response(new Date().toISOString());
};
```

### In Server Mode (opt-in to prerendering)

```astro
---
// src/pages/about.astro
export const prerender = true; // Static at build time
---

<h1>About Us</h1>
```

## Available Adapters

### Node.js

```bash
npx astro add node
```

```javascript
// astro.config.mjs
import node from '@astrojs/node';

export default defineConfig({
  output: 'server',
  adapter: node({
    mode: 'standalone', // or 'middleware' (mount in Express/Fastify)
    // staticHeaders: true,  // send CSP etc. as headers for prerendered pages
    // bodySizeLimit: ...,   // max request body size
  }),
});
```

### Vercel

```bash
npx astro add vercel
```

```javascript
// astro.config.mjs
import vercel from '@astrojs/vercel';

export default defineConfig({
  adapter: vercel({
    webAnalytics: { enabled: true },
    imageService: true,        // use Vercel Image Optimization
    isr: true,                 // cache on-demand pages like static ones after first request
    // isr: { expiration: 86400, exclude: ['/preview', '/auth/[page]'] },
    // middlewareMode: 'edge', // run middleware in front of the ISR cache (on every request)
    // maxDuration: 30, skewProtection: true, staticHeaders: true,
  }),
});
```

Vercel notes: ISR requests **don't include search params**, and by default middleware runs only on a cache miss or regeneration. Sessions need an explicit driver (e.g. Redis from the Vercel marketplace + `ioredis`).

### Netlify

```bash
npx astro add netlify
```

```javascript
// astro.config.mjs
import netlify from '@astrojs/netlify';

export default defineConfig({
  adapter: netlify({
    middlewareMode: 'edge', // run middleware on Edge Functions for ALL requests (incl. prerendered pages)
  }),
});
```

### Cloudflare

```bash
npx astro add cloudflare
```

```javascript
// astro.config.mjs
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  adapter: cloudflare({
    imageService: 'cloudflare-binding', // default since v13 (was 'compile')
    // sessionKVBindingName: 'SESSION',  // sessions use KV automatically
    // prerenderEnvironment: 'node',     // if prerendered pages need Node APIs
  }),
});
```

**Cloudflare adapter v13+ (Astro 6/7) — breaking vs v12:**
- Targets **Workers only** (Pages support removed). `mode`, `routes`, `cloudflareModules`, `workerEntryPoint` options are gone.
- `astro dev` / `astro preview` run in the real `workerd` runtime via the Cloudflare Vite plugin.
- `Astro.locals.runtime` was **removed**. Read bindings/vars with `import { env } from 'cloudflare:workers'` (or typed via `astro:env/server`). Execution context: `Astro.locals.cfContext.waitUntil(...)`; geo data: `Astro.request.cf`.
- `wrangler.jsonc` is optional for simple projects; when present, `"main": "@astrojs/cloudflare/entrypoints/server"`.
- Deploying a named environment: `CLOUDFLARE_ENV=staging astro build`.
- Route caching (Astro 7): `import { cacheCloudflare } from '@astrojs/cloudflare/cache'` (experimental CDN provider).

### Deno

The adapter is maintained by Deno now, not Astro: install `@deno/astro-adapter` and follow the [denoland/deno-astro-adapter](https://github.com/denoland/deno-astro-adapter) README. `@astrojs/deno` is obsolete.

## SSR Features

### Request Object

```astro
---
// Available in SSR pages
const url = Astro.url;
const method = Astro.request.method;
const headers = Astro.request.headers;
const userAgent = headers.get('user-agent');

// Get request body (POST, PUT, etc.)
if (method === 'POST') {
  const formData = await Astro.request.formData();
  const json = await Astro.request.json();
}
---
```

### Cookies

```astro
---
// Reading cookies
const sessionId = Astro.cookies.get('session')?.value;
const prefs = Astro.cookies.get('prefs')?.json();

// Setting cookies
Astro.cookies.set('session', 'abc123', {
  path: '/',
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 60 * 60 * 24 * 7, // 1 week
});

// Deleting cookies
Astro.cookies.delete('session');
---
```

### Response Headers

```astro
---
Astro.response.headers.set('Cache-Control', 'max-age=3600');
Astro.response.headers.set('X-Custom-Header', 'value');
---
```

### Redirects

```astro
---
if (!user) {
  return Astro.redirect('/login', 302);
}
---
```

## Server Islands

Defer rendering of specific Astro components to the server. Each island loads independently, keeping main content fast. No experimental flag needed.

```astro
---
import UserProfile from '../components/UserProfile.astro';
import ProductReviews from '../components/ProductReviews.astro';
---

<!-- Static content renders immediately -->
<h1>Welcome</h1>

<!-- Server-rendered on each request -->
<UserProfile server:defer>
  <p slot="fallback">Loading profile...</p>
</UserProfile>

<!-- Multiple islands load in parallel -->
<ProductReviews server:defer productId="abc-123">
  <div slot="fallback">Loading reviews...</div>
</ProductReviews>
```

Key points:
- Requires an adapter (same as SSR)
- Props must be serializable (no functions)
- Use `Referer` header to access page URL inside island
- See [references/server-islands.md](server-islands.md) for full details

## Sessions

Server-side session storage for on-demand rendered pages (Astro 5.7+):

```javascript
// astro.config.mjs
import { defineConfig, sessionDrivers } from 'astro/config';

export default defineConfig({
  adapter: node({ mode: 'standalone' }),
  session: {
    driver: sessionDrivers.redis({ url: process.env.REDIS_URL }),
  },
});
```

```astro
---
export const prerender = false;
const cart = await Astro.session?.get('cart');
await Astro.session?.set('lastVisit', new Date());
---
```

See [references/sessions.md](sessions.md) for full details.

## API Endpoints

### GET, POST, PUT, DELETE

```typescript
// src/pages/api/users/[id].ts
import type { APIRoute } from 'astro';

export const GET: APIRoute = async ({ params, request }) => {
  const { id } = params;
  const user = await db.users.findById(id);

  if (!user) {
    return new Response(null, { status: 404 });
  }

  return new Response(JSON.stringify(user), {
    headers: { 'Content-Type': 'application/json' },
  });
};

export const PUT: APIRoute = async ({ params, request }) => {
  const { id } = params;
  const data = await request.json();

  await db.users.update(id, data);

  return new Response(null, { status: 204 });
};

export const DELETE: APIRoute = async ({ params }) => {
  const { id } = params;
  await db.users.delete(id);
  return new Response(null, { status: 204 });
};
```

### Streaming Responses

```typescript
// src/pages/api/stream.ts
import type { APIRoute } from 'astro';

export const GET: APIRoute = async () => {
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      for (let i = 0; i < 10; i++) {
        controller.enqueue(encoder.encode(`data: ${i}\n\n`));
        await new Promise(r => setTimeout(r, 1000));
      }
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  });
};
```

## Context and Locals

Access shared data across middleware and pages:

```typescript
// src/middleware.ts
export const onRequest = async ({ locals, request }, next) => {
  const token = request.headers.get('authorization');
  locals.user = await validateToken(token);
  return next();
};
```

```astro
---
// src/pages/dashboard.astro
const { user } = Astro.locals;

if (!user) {
  return Astro.redirect('/login');
}
---

<h1>Welcome, {user.name}</h1>
```

## Best Practices

1. **Stay static by default** - `output: 'static'` + adapter, `prerender = false` only where needed
2. **Prerender where possible** - Better performance and caching
3. **Use appropriate adapter** - Match your deployment platform
4. **Handle errors gracefully** - Return proper status codes
5. **Cache on-demand routes** - `Astro.cache.set()` / `routeRules` in Astro 7 (see caching-and-fetch.md), or manual `Cache-Control` headers
6. **Validate user input** - Never trust request data
7. **Use locals for shared state** - Cleaner than passing through props
