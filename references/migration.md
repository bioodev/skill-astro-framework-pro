# Version Migration Checklist (Astro 5 → 6 → 7)

Check `package.json` first. Current stable line: **Astro 7.x** (7.0 released 2026-06). Official guides: `https://docs.astro.build/en/guides/upgrade-to/v6/` and `/v7/`. Run `npx @astrojs/upgrade` to bump Astro and official integrations together.

## Contents
- Symptom → cause lookup
- Astro 6 changes
- Astro 7 changes

---

## Symptom → Cause

| Symptom after upgrade | Cause | Fix |
|---|---|---|
| Spaces missing between inline elements (`hello</span><em>world`) | v7 `compressHTML: 'jsx'` default | `compressHTML: true` or add `{" "}` |
| Build error on unclosed/invalid HTML that "used to work" | v7 Rust compiler is strict, no auto-correction | Fix the markup |
| remark/rehype plugins silently ignored | v7 Sätteri markdown processor | `npm i @astrojs/markdown-remark`; `markdown: { processor: unified() }` |
| `src/fetch.ts` behaves as a router | v7 reserved entrypoint | Rename file or set `fetchFile` |
| `@astrojs/db` / `astro:db` missing | v7 removed Astro DB | Drizzle + libSQL/Turso, `node:sqlite`, Neon, etc. |
| `entry.render is not a function`, `slug` undefined | v6 removed legacy collections | `render(entry)`, `entry.id` |
| `Astro.glob is not a function` | v6 removal | `Object.values(import.meta.glob('./x/*.md', { eager: true }))` or a collection |
| `ViewTransitions` not exported | v6 removal | `import { ClientRouter } from 'astro:transitions'` |
| `z.string().email()` deprecation / type errors | Zod 4 | `z.email()`, `z.url()`, `{ error: '...' }` |
| `if (import.meta.env.FLAG === true)` never true | v6 env values are inlined as **strings** | Compare to `"true"` or use `astro:env` with `envField.boolean` |
| Secrets undefined at runtime in SSR | v6 no longer rewrites non-public vars to `process.env` | Use `astro:env/server` (`getSecret`) or `process.env` explicitly |
| Images now cropped / not upscaled | v6 image defaults | Pass `fit` explicitly if you need the old behavior |
| `/sitemap.xml/` 404 | v6 endpoints with extension have no trailing-slash variant | Link without trailing slash |
| Default-locale URLs no longer redirect | v6 `redirectToDefaultLocale: false` default | Set it `true` (requires `prefixDefaultLocale: true`) |
| `Astro.locals.runtime` undefined on Cloudflare | Cloudflare adapter v13 | `import { env } from 'cloudflare:workers'` |
| Node 18/20 CI fails | v6 requires Node ≥ 22.12 | Upgrade CI runtime |

---

## Astro 6 (2026-03)

**Platform:** Node ≥ 22.12 · Vite 7 · Zod 4 · Shiki 4 · config files `.mjs/.js/.ts/.mts` only (no `.cjs`).

**Removed:** legacy content collections (`src/content/config.ts`, `type`, `slug`, `entry.render()`, `getEntryBySlug`, `getDataEntryById`) · `<ViewTransitions />` · `Astro.glob()` · `emitESMImage()` (→ `emitImageMetadata()`) · `astro:ssr-manifest` (→ `astro:config/server`) · `<ClientRouter handleForms>` (now default) · `prefetch(url, { with })` · session `test` driver · loader `schema` as function (→ `createSchema()`) · `%25` in route filenames.

**Deprecated:** `z` from `astro:content`/`astro:schema` (→ `astro/zod`) · `Astro.*` inside `getStaticPaths()` (→ `import.meta.env.SITE`) · `import.meta.env.ASSETS_PREFIX` (→ `build.assetsPrefix` from `astro:config/server`) · session driver strings (→ `sessionDrivers.redis({...})`).

**Stabilized (remove the `experimental` flags):** CSP → `security.csp` · Fonts API → top-level `fonts` · live content collections · `preserveScriptOrder` (scripts/styles now render in declaration order) · `staticImportMetaEnv` · `headingIdCompat` · `failOnPrerenderConflict` → `prerenderConflictBehavior`.

**Temporary escape hatch:** `legacy: { collectionsBackwardsCompat: true }`.

## Astro 7 (2026-06)

**Platform:** Vite 8 · Rust compiler (only compiler; stricter HTML) · Sätteri markdown by default · queued rendering (~2× faster).

**Breaking:** `compressHTML: 'jsx'` default · `src/fetch.ts` reserved · `@astrojs/db` removed · remaining `astro:transitions` internals removed (use event names like `'astro:before-preparation'`).

**Stabilized (move out of `experimental`):** `cache` and `routeRules` (route caching) · `advancedRouting` · `logger` · `rustCompiler` · `queuedRendering`.

**Deprecated:** `getContainerRenderer` from `@astrojs/<framework>` → `@astrojs/<framework>/container-renderer`.

**Agent tooling:** `astro dev` auto-backgrounds when it detects an AI agent (disable with `ASTRO_DEV_BACKGROUND=0`); health check at `/_astro/status`; `astro dev --json` for structured logs.
