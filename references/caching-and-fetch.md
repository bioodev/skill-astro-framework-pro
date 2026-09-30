# Route Caching & Advanced Routing (Astro 7)

Both were experimental in Astro 6 and are **stable, top-level config in Astro 7**. If a project still has `experimental.cache`, `experimental.routeRules` or `experimental.advancedRouting`, move/remove them.

## Contents
- Route caching: config, `Astro.cache`, `routeRules`, invalidation, CDN providers, limits
- Advanced routing: `src/fetch.ts`, `astro/fetch` handlers, Hono composition

---

## Route Caching

Only applies to **on-demand rendered** routes (`prerender = false` or `output: 'server'`). Prerendered pages are already static files.

### Config

```javascript
// astro.config.mjs
import { defineConfig, memoryCache } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  adapter: node({ mode: 'standalone' }),
  cache: { provider: memoryCache() },

  // Declarative per-pattern rules (static, [param], [...rest] patterns)
  routeRules: {
    '/api/[...path]': { swr: 600 },
    '/products/[id]': { maxAge: 3600, tags: ['products'] },
    '/blog/[...slug]': { maxAge: 300, swr: 60 },
  },
});
```

CDN providers (experimental, need manual enablement):

| Host | Import | Adapter version |
|---|---|---|
| Netlify | `import { cacheNetlify } from '@astrojs/netlify/cache'` | ≥ 8.0.0 |
| Vercel | `import { cacheVercel } from '@astrojs/vercel/cache'` | ≥ 11.0.0 |
| Cloudflare | `import { cacheCloudflare } from '@astrojs/cloudflare/cache'` | ≥ 14.0.0 |

### In routes: `Astro.cache` / `context.cache`

```astro
---
export const prerender = false;
const product = await getProduct(Astro.params.id!);

Astro.cache.set({
  maxAge: 120,          // seconds fresh
  swr: 60,              // serve stale while revalidating
  tags: ['products', `product:${product.id}`],
  lastModified: product.updatedAt,
  // etag: '...'
});
---
```

- `Astro.cache.set(false)` opts the route out.
- Multiple `set()` calls merge: scalars last-write-wins, `lastModified` keeps the latest date, `tags` accumulate.
- Read state: `cache.options`, `cache.tags`, `cache.enabled` (false in dev or without a provider).
- **Dev mode:** the API exists (no conditionals needed) but nothing is cached — verify caching with `astro build && astro preview` or on the host.

### Invalidation

```typescript
// src/pages/api/products/[id].ts — after a mutation
export const prerender = false;
export async function POST(context) {
  await updateProduct(context.params.id, await context.request.json());
  await context.cache.invalidate({ tags: ['products'] });
  await context.cache.invalidate({ path: `/products/${context.params.id}` }); // exact match, no wildcards
  return Response.json({ purged: true });
}
```

### With live collections

`getLiveCollection()` / `getLiveEntry()` return a `cacheHint` (`tags`, `lastModified`). Forward it: `if (cacheHint) Astro.cache.set(cacheHint);`

### Decision

```
Page identical for everyone and changes per deploy → prerender (no cache API needed)
Same for everyone but data changes hourly/daily → on-demand + maxAge/swr + tags, invalidate on write
Personalized per user → don't cache the page; cache the static shell and use server:defer for the personal part
```

---

## Advanced Routing: `src/fetch.ts`

Astro 7 treats `src/fetch.ts` as a **reserved entrypoint** that wraps the whole request pipeline. If a project already has an unrelated `src/fetch.ts` (e.g. an API client), rename it or set `fetchFile: null` / `fetchFile: 'handler'` in config.

```typescript
// src/fetch.ts — keep Astro routing, add logic around it
import { FetchState, astro } from 'astro/fetch';

export default {
  async fetch(request: Request): Promise<Response> {
    const state = new FetchState(request);
    const response = await astro(state);
    response.headers.set('X-Powered-By', 'Astro');
    return response;
  },
};
```

Compose individual stages instead of the full `astro()` pipeline (available: `astro`, `actions`, `cache`, `i18n`, `middleware`, `pages`, `redirects`, `sessions`, `trailingSlash`):

```typescript
import { FetchState, actions, middleware, pages, i18n } from 'astro/fetch';

export default {
  async fetch(request: Request): Promise<Response> {
    const state = new FetchState(request);

    const actionResponse = await actions(state);
    if (actionResponse) return actionResponse;

    // custom logic between actions and page rendering
    const response = await middleware(state, (s) => pages(s));
    return i18n(state, response);
  },
};
```

Type the default export with `satisfies Fetchable` (`import type { Fetchable } from 'astro'`).

It also composes with **Hono** (auth, logging, CORS middleware) when you need full control of the pipeline.

**When to use it vs `src/middleware.ts`:** middleware (`onRequest`) is enough for auth checks, `locals`, redirects and headers. Reach for `src/fetch.ts` only when you need to reorder/replace Astro's pipeline stages or mount another framework in front of Astro.
