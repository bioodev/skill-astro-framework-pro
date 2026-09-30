---
name: astro-framework-pro
description: Astro (v5–v7, current 7.x) specialist for content-driven sites with islands architecture. Use when creating or reviewing Astro components, choosing static vs on-demand rendering, hydration directives (client:load/idle/visible/media/only), server islands (server:defer), content collections (Content Layer loaders, live collections), route caching (Astro.cache, routeRules), src/fetch.ts advanced routing, actions, middleware, sessions, astro:env, i18n, view transitions (ClientRouter), images/fonts, adapters (Node, Vercel, Netlify, Cloudflare), integrating React/Vue/Svelte/Solid islands, or upgrading an Astro project between major versions. Triggers on astro.config.*, .astro files, content.config.ts, or mentions of Astro. Not for full-SPA frameworks (Next.js, Remix, SvelteKit).
license: MIT
compatibility: Astro 5.x–7.x (verified against 7.3.5). Node >= 22.12 for Astro 6+.
metadata:
  author: bioodev
  version: "1.0.0"
  created: "2026-09-30"
  last-verified: "2026-09-30"
  verified-against: "astro@7.3.5, @astrojs/cloudflare@14.3.3"
  sources: "docs.astro.build, context7 /withastro/docs"
  based-on: "delineas/astro-framework-agents (MIT), incluud/astro-agent-skills (MIT), oakoss/agent-skills (MIT)"
  tags: "astro, islands, ssr, content-collections, live-collections, route-caching, server-islands, actions, sessions, view-transitions, i18n, cloudflare"
---

# Astro Framework

Build fast, content-first Astro sites: static HTML by default, JavaScript only where it earns its place.

## Step 0: Check the version before writing code

Read `package.json` → `astro` version. Astro changed a lot across 5 → 6 → 7. Most bad advice comes from mixing up versions.

| Installed | Key facts |
|---|---|
| **7.x** (current) | Rust compiler (strict HTML) · `compressHTML: 'jsx'` default · Sätteri markdown (remark/rehype need `@astrojs/markdown-remark`) · stable `cache`/`routeRules` · `src/fetch.ts` reserved · **no Astro DB** |
| **6.x** | Node ≥ 22.12 · Zod 4 · legacy collections, `Astro.glob`, `<ViewTransitions />` removed · CSP/Fonts/live collections stable |
| **5.x** | Content Layer + legacy compat · `hybrid` already gone · sessions from 5.7 |

Upgrading, or seeing errors after an upgrade? Load [references/migration.md](references/migration.md).
This skill is self-contained for everyday work. It covers Astro through 7.3. Consult external sources only in the cases listed under **External Sources** at the end.

## Decision Frameworks

### Rendering mode

`output: 'hybrid'` **no longer exists** (removed in Astro 5). There are only two modes:

```
output: 'static' (default) — every route prerendered unless it opts out
├── Blog, docs, marketing, portfolio → no adapter needed
└── Mostly static + some dynamic routes (login, cart, API)
    → add an adapter + `export const prerender = false` on those routes only
    → prefer server:defer islands over making a whole page on-demand

output: 'server' — every route on-demand unless `prerender = true`
└── Only when >80% of routes need request data (auth-gated SaaS/dashboard)
```

**Signs the choice is wrong:** builds >10 min from huge `getStaticPaths` → move those routes on-demand with route caching. More than half the routes have `prerender = false` → switch to `'server'`. `'server'` when only 2 pages read cookies → go back to static + opt-outs.

### Static vs server island vs client island

```
Needs per-request server data (session, cookies, DB, personalization)?
├── Yes → server:defer (with slot="fallback")
│         e.g. avatar, cart count, price/stock on an otherwise static product page
└── No → Needs browser interactivity (state, onClick, onChange)?
    ├── Yes → framework component + client:* directive
    └── No  → plain .astro, zero JS (this should be ~90% of the site)
```

The e-commerce pattern shows all three strategies on one page. The product page is static. Price and stock use `server:defer`. The add-to-cart button uses `client:load`.

### Hydration directive

| Directive | Use for | Trap |
|---|---|---|
| `client:load` | Above-the-fold, needed immediately | Big bundles (>50KB): hydrate only the interactive leaf |
| `client:idle` | Non-urgent widgets | Idle can fire seconds late on busy/low-end devices. Cap it with `client:idle={{ timeout: 500 }}` or use `load` |
| `client:visible` | Below the fold (`{{ rootMargin: '200px' }}` to hydrate early) | Pointless on hero/header: it's visible at load anyway, so use `load` |
| `client:media="(q)"` | Device-specific UI (mobile menu) | — |
| `client:only="react"` | Needs browser APIs, no SSR possible | The framework string is **required** |

Don't hydrate a whole navbar for a menu toggle. Use a vanilla `<script>` in the `.astro` component instead. Islands can't share React context; use nanostores (see client-directives.md).

### Content data source

```
Local md/mdx/json/yaml files      → glob() loader in src/content.config.ts
One JSON/YAML/TOML file           → file() loader
CMS/API, fresh per deploy         → custom (object) loader, build-time
CMS/API, fresh per request        → live collection in src/live.config.ts (+ route caching)
```

### Caching on-demand routes (Astro 7)

Same for everyone, changes occasionally → `Astro.cache.set({ maxAge, swr, tags })` or `routeRules`, and invalidate by tag on writes. Personalized → don't cache the page; use a static shell + `server:defer`. See [references/caching-and-fetch.md](references/caching-and-fetch.md).

### When NOT to use Astro

Full SPA with client-side routing and heavy global state, real-time collaboration at the core, or everything behind auth with no public content. Say so, and suggest a SPA framework instead.

## Critical Rules

**Always:**
- Import Zod from `astro/zod` (Zod 4: `z.email()`, `z.url()`, `{ error: '...' }`). Use `z.coerce.date()` for frontmatter dates. Give booleans a `.default()`.
- Use `import { render } from 'astro:content'` → `await render(entry)`, and `entry.id` for route params
- Use `<Image />` / `<Picture />` from `astro:assets` with imported local images, and `image()` in collection schemas. Always write alt text.
- Type env vars with the `astro:env` schema (`envField`). Secrets come from `astro:env/server`.
- Type component props with `interface Props`. Type `App.Locals` in `src/env.d.ts`.
- Use `sessionDrivers.<name>({...})` for session config, not string drivers
- Use `<ClientRouter />` (from `astro:transitions`) in the shared layout's `<head>` for SPA-like navigation
- Install an adapter whenever any route is on-demand (`npx astro add <adapter>`). Use `astro add` for official integrations.

**Never:**
- `output: 'hybrid'`, `entry.render()`, `entry.slug`, `Astro.glob()`, `<ViewTransitions />`, `src/content/config.ts`, `type: 'content'`. All removed.
- `z` from `astro:content`/`astro:schema` (deprecated). `Astro.*` inside `getStaticPaths()` (use `import.meta.env.SITE`).
- `@astrojs/db` on Astro 7 (removed). Use Drizzle + libSQL/Turso/Neon or `node:sqlite`.
- `Astro.locals.runtime` on Cloudflare adapter v13+. Use `import { env } from 'cloudflare:workers'`.
- `client:*` on `.astro` components (they never hydrate), or passing functions or class instances as island props
- Browser APIs (`window`, `document`) in frontmatter. It runs on the server.
- `Astro.request` headers/cookies or `Astro.session` in prerendered pages
- `import.meta.env.X === true`. Since v6, env values are inlined as strings.
- Importing `astro:content` in client code. Pass data as props.
- `security.csp` together with `<ClientRouter />` or Shiki (both unsupported under CSP)
- Naming an unrelated file `src/fetch.ts` on Astro 7 (it's the routing entrypoint)

## Common Mistakes → Fix

| Mistake | Fix |
|---|---|
| `client:load` everywhere | No directive by default. Then `visible`/`idle`/`load`, chosen per component. |
| Whole page on-demand for one personalized widget | Static page + `server:defer` widget |
| `<ClientRouter />` only on some pages | Put it in the shared layout |
| Scripts stop working after navigation | Listen for `astro:page-load`, or use `data-astro-rerun` |
| `transition:persist` state lost | The element must exist on both pages with the same persist name |
| Missing spaces between inline tags (v7) | `compressHTML: true` or `{" "}` |
| remark/rehype plugins ignored (v7) | `markdown: { processor: unified({ remarkPlugins, rehypePlugins }) }` from `@astrojs/markdown-remark` |
| Middleware not running | Must be `src/middleware.ts` or `src/middleware/index.ts`. Chain with `sequence()`. |
| "Protected" static page is public | Middleware runs at build time for prerendered pages. Protected pages need `prerender = false`. Use `context.isPrerendered` to skip request-only logic. |
| `z` / `getActionResult` imported from `astro:actions` | Neither is exported there: use `z` from `astro/zod` and `Astro.getActionResult()` |
| `getCollection()` without `await` | Always await collection queries |
| Tailwind via `@astrojs/tailwind` | Tailwind v4: `npx astro add tailwind` (uses `@tailwindcss/vite`) |

## Reference Documentation

Load only what the current task needs:

| Topic | File | Load when |
|---|---|---|
| Version upgrades | [references/migration.md](references/migration.md) | Upgrading, errors after an upgrade, unsure which API applies |
| Components | [references/components.md](references/components.md) | Props, slots, expressions, `class:list` |
| Hydration & islands | [references/client-directives.md](references/client-directives.md) | `client:*`, shared state (nanostores), serializable props |
| Server islands | [references/server-islands.md](references/server-islands.md) | `server:defer`, fallbacks |
| Content collections | [references/content-collections.md](references/content-collections.md) | Loaders, schemas, queries, live collections |
| Routing | [references/routing.md](references/routing.md) | Pages, dynamic routes, endpoints, redirects |
| Caching & fetch.ts | [references/caching-and-fetch.md](references/caching-and-fetch.md) | Astro 7 `Astro.cache`, `routeRules`, invalidation, `src/fetch.ts` |
| SSR & adapters | [references/ssr-adapters.md](references/ssr-adapters.md) | On-demand rendering, Node/Vercel/Netlify/Cloudflare |
| Sessions | [references/sessions.md](references/sessions.md) | `Astro.session`, carts, drivers |
| Actions | [references/actions.md](references/actions.md) | `defineAction`, forms, validation |
| Middleware | [references/middleware.md](references/middleware.md) | `onRequest`, `sequence`, `locals` |
| Env variables | [references/environment-variables.md](references/environment-variables.md) | `astro:env`, `envField`, secrets |
| View transitions | [references/view-transitions.md](references/view-transitions.md) | `ClientRouter`, animations, prefetch, lifecycle |
| Images | [references/images.md](references/images.md) | `<Image>`/`<Picture>`, responsive `layout`, `priority`, SVG components, remote domains |
| Styling | [references/styling.md](references/styling.md) | Scoped CSS, globals, Tailwind |
| Configuration | [references/configuration.md](references/configuration.md) | `astro.config`, TS, experimental flags, Markdown processors (Sätteri/unified), Vite |
| i18n | [references/i18n-routing.md](references/i18n-routing.md) | Locales, `astro:i18n` helpers, fallback |
| A11y & review | [references/accessibility-and-review.md](references/accessibility-and-review.md) | **Before finishing any task**, or when reviewing a project |

## Related Skills

- **SEO** (meta tags, JSON-LD, sitemaps, OG images, IndexNow, llms.txt): hand off to the `astro-seo` skill if it's installed
- **Cloudflare deployment details:** the `cloudflare:*` skills; **Tailwind layouts:** `tailwindcss-advanced-layouts`

## Output Format

When implementing, provide:
1. The files (`.astro`, `content.config.ts`, `astro.config.*` changes), with full paths
2. Prop and data types
3. One line explaining the rendering and hydration choice ("static + `server:defer` for stock because…")
4. Verification: `npx astro check && npx astro build`. On Astro 7, `astro dev` runs in the background when an agent is detected, with a health check at `/_astro/status`.

## Essential API Summary (Astro 7)

The minimum correct syntax for each area. For depth, load the matching reference.

**Component**
```astro
---
interface Props { title: string; tags?: string[] }
const { title, tags = [] } = Astro.props;       // frontmatter runs on the server only
---
<article class:list={['card', { featured: tags.includes('top') }]}>
  <h2>{title}</h2><slot /><slot name="footer" />
</article>
<style>h2 { color: var(--accent); }</style>      <!-- scoped by default -->
<script>/* bundled client JS, runs once per page load */</script>
```

**Collections**: `src/content.config.ts`
```ts
import { defineCollection, reference } from 'astro:content';
import { glob, file } from 'astro/loaders';
import { z } from 'astro/zod';
const blog = defineCollection({
  loader: glob({ base: './src/content/blog', pattern: '**/*.{md,mdx}' }),
  schema: ({ image }) => z.object({
    title: z.string(), pubDate: z.coerce.date(), draft: z.boolean().default(false),
    cover: image().optional(), author: reference('authors'),
  }),
});
const authors = defineCollection({ loader: file('src/data/authors.json'), schema: z.object({ name: z.string(), email: z.email() }) });
export const collections = { blog, authors };
```

**Dynamic route + render**: `src/pages/blog/[...slug].astro`
```astro
---
import { getCollection, render } from 'astro:content';
export async function getStaticPaths() {
  const posts = await getCollection('blog', ({ data }) => !data.draft);
  return posts.map((post) => ({ params: { slug: post.id }, props: { post } }));
}
const { post } = Astro.props;
const { Content, headings } = await render(post);
---
<Content />
```

**Endpoint**: `src/pages/api/items.ts`
```ts
import type { APIRoute } from 'astro';
export const prerender = false;
export const GET: APIRoute = async ({ url, cookies, locals }) => Response.json({ ok: true });
```

**Action**: `src/actions/index.ts`
```ts
import { defineAction, ActionError } from 'astro:actions';
import { z } from 'astro/zod';
export const server = {
  subscribe: defineAction({
    accept: 'form',                                   // default 'json'
    input: z.object({ email: z.email() }),
    handler: async ({ email }, context) => { /* throw new ActionError({ code: 'UNAUTHORIZED' }) */ return { ok: true }; },
  }),
};
// client: import { actions } from 'astro:actions'; const { data, error } = await actions.subscribe(formData);
// HTML form (page must be prerender = false): <form method="POST" action={actions.subscribe}> + Astro.getActionResult(actions.subscribe)
// from server code: await Astro.callAction(actions.subscribe, input) · field errors: isInputError(error) && error.fields
```

**Middleware**: `src/middleware.ts`
```ts
import { defineMiddleware, sequence } from 'astro:middleware';
const auth = defineMiddleware(async (ctx, next) => {
  ctx.locals.user = await getUser(ctx.cookies.get('sid')?.value);   // mutate locals, don't replace it
  if (ctx.url.pathname.startsWith('/admin') && !ctx.locals.user) return ctx.redirect('/login');
  return next();
});
export const onRequest = sequence(auth);
```

**Env, sessions, cache**: `astro.config.mjs`
```js
import { defineConfig, envField, sessionDrivers, memoryCache, fontProviders } from 'astro/config';
export default defineConfig({
  site: 'https://example.com',                      // required for canonicals/sitemap
  env: { schema: {
    PUBLIC_API_URL: envField.string({ context: 'client', access: 'public' }),
    API_SECRET: envField.string({ context: 'server', access: 'secret' }),
  } },                                              // import from 'astro:env/client' | 'astro:env/server'
  session: { driver: sessionDrivers.redis({ url: process.env.REDIS_URL }) }, // some adapters provide a default
  cache: { provider: memoryCache() },
  routeRules: { '/products/[id]': { maxAge: 3600, swr: 60, tags: ['products'] } },
  fonts: [{ provider: fontProviders.fontsource(), name: 'Inter', cssVariable: '--font-inter' }],
  security: { csp: true },   // build/preview only; NOT compatible with <ClientRouter /> or Shiki (use <Prism />)
});
```
In routes: `await Astro.session?.get('cart')` / `.set()`, `Astro.cache.set({ maxAge, tags })`, `context.cache.invalidate({ tags })`. On `@astrojs/node`, set `staticHeaders: true` so prerendered pages send the CSP as a header.

**Islands, images, navigation, i18n**
```astro
---
import { Image, Picture, Font } from 'astro:assets';
import { ClientRouter } from 'astro:transitions';
import { getRelativeLocaleUrl } from 'astro:i18n';
import hero from '../assets/hero.jpg';
import Counter from '../components/Counter.tsx';
import Avatar from '../components/Avatar.astro';
---
<head><ClientRouter /><Font cssVariable="--font-inter" preload /></head>
<Image src={hero} alt="Product on a desk" priority />   <!-- local import: width/height inferred -->
<Counter client:visible initial={3} />                  <!-- serializable props only -->
<Avatar server:defer><span slot="fallback">…</span></Avatar>
<a href={getRelativeLocaleUrl('es', 'about')}>ES</a>
<a href="/docs" data-astro-prefetch="hover">Docs</a>
```

**CLI:** `npm create astro@latest` · `npx astro add <integration|adapter>` · `astro dev` · `astro check` · `astro build` · `astro preview` · `npx @astrojs/upgrade`

## External Sources (optional)

The skill contains the fundamentals. Go outside it **only** when:
- the installed version is newer than 7.3, or an API isn't covered here or in `references/`
- behavior contradicts the skill (then trust the official docs, and note the discrepancy)
- you need implementation details or internals to debug

| Source | Use for | Notes |
|---|---|---|
| **context7** (`mcp__context7__*`) | Current API usage, config options, upgrade guides | Library `/withastro/docs` (official docs, best choice). `/websites/astro_build_en` as an alternative |
| **Astro Docs MCP** (`https://mcp.docs.astro.build/mcp`) | Same as above, official source | Only if it's configured |
| **DeepWiki** (`mcp__deepwiki__*`) | **Internals** of `withastro/astro`: build pipeline, SSR runtime, how an adapter works | Its index can lag behind releases. Don't use it to decide which API is current |
| **docs.astro.build** (WebFetch) | Upgrade guides `/en/guides/upgrade-to/v6/`, `/v7/` | Fallback if the MCPs aren't available |
