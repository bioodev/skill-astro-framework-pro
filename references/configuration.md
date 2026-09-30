# Configuration

Astro configuration lives in `astro.config.mjs` at the project root.

## Contents
- Basic Configuration
- Common Options
- Vite Configuration
- Markdown Configuration
- i18n Configuration
- Image Configuration
- Redirects
- TypeScript Configuration
- Environment Variables
- Session Configuration (Astro 5.7+)
- Experimental Features
- Full Example
- Best Practices

## Basic Configuration

```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';

export default defineConfig({
  // Your configuration options here
});
```

## Common Options

### Site URL

```javascript
export default defineConfig({
  site: 'https://example.com',
  base: '/blog', // For subdirectory deployments
});
```

### Output Mode

```javascript
import node from '@astrojs/node';

export default defineConfig({
  output: 'static',    // Default - all pages prerendered
  // output: 'server', // All pages server-rendered
  // ('hybrid' was removed in Astro 5: stay 'static' + adapter + per-route `prerender = false`)

  adapter: node({      // Required for any on-demand route
    mode: 'standalone',
  }),
});
```

### Integrations

```javascript
import react from '@astrojs/react';
import tailwindcss from '@tailwindcss/vite';
import mdx from '@astrojs/mdx';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  integrations: [react(), mdx(), sitemap()],
  vite: { plugins: [tailwindcss()] }, // Tailwind v4 is a Vite plugin, not an integration
});
```

### Build Options

```javascript
export default defineConfig({
  build: {
    format: 'directory',  // /about/index.html (default)
    // format: 'file',    // /about.html

    inlineStylesheets: 'auto', // Inline small stylesheets
    // inlineStylesheets: 'always',
    // inlineStylesheets: 'never',

    assets: '_astro',     // Assets directory name
  },

  compressHTML: 'jsx',    // Astro 7 default: JSX-style whitespace (strips spaces between inline tags!)
  // compressHTML: true,  // Pre-v7 HTML-aware behavior — set this if inline elements lose spaces

  outDir: './dist',       // Build output directory
  publicDir: './public',  // Static assets directory
});
```

### Dev Server

```javascript
export default defineConfig({
  server: {
    port: 4321,           // Default port
    host: true,           // Expose to network
    open: true,           // Open browser on start
  },

  devToolbar: {
    enabled: true,        // Show dev toolbar
  },
});
```

### Prefetching

```javascript
export default defineConfig({
  prefetch: {
    prefetchAll: true,              // Prefetch all links
    defaultStrategy: 'viewport',    // 'hover' | 'viewport' | 'load'
  },
});
```

## Vite Configuration

```javascript
export default defineConfig({
  vite: {
    plugins: [],
    resolve: {
      alias: {
        '@': '/src',
        '@components': '/src/components',
      },
    },
    css: {
      preprocessorOptions: {
        scss: {
          additionalData: `@use "@/styles/variables" as *;`, // Sass: @use, not the deprecated @import
        },
      },
    },
  },
});
```

> Astro 7 runs on **Vite 8** (Rolldown-based bundler). Custom `vite.build.rollupOptions` / `manualChunks` tweaks from older projects may behave differently. Check the Vite 8 migration guide before copying them over, and prefer Astro's defaults unless you've measured a problem.

## Markdown Configuration

Syntax highlighting stays at the top level. Everything else goes **to the processor** (Astro 6.4+). Top-level `markdown.remarkPlugins`, `rehypePlugins`, `remarkRehype`, `gfm` and `smartypants` are **deprecated**.

```javascript
import { defineConfig } from 'astro/config';
import { satteri } from '@astrojs/markdown-satteri';   // npm i @astrojs/markdown-satteri (only needed to configure it)

export default defineConfig({
  markdown: {
    syntaxHighlight: 'shiki',              // 'shiki' | 'prism' | false. Use 'prism' with security.csp
    shikiConfig: { theme: 'github-dark', wrap: true },
    processor: satteri({                   // Astro 7 default processor (Rust)
      features: { gfm: true, math: true, directive: true },
      // mdastPlugins: [...], hastPlugins: [...]   // Sätteri-native plugins
    }),
  },
});
```

Keeping remark/rehype plugins (e.g. `remark-toc`, `rehype-slug`):

```javascript
import { unified, rehypeHeadingIds } from '@astrojs/markdown-remark';  // npm i @astrojs/markdown-remark
import remarkToc from 'remark-toc';

export default defineConfig({
  markdown: {
    processor: unified({
      remarkPlugins: [remarkToc],
      rehypePlugins: [rehypeHeadingIds],
      // gfm: false, smartypants: false, remarkRehype: { ... }
    }),
  },
});
```

## i18n Configuration

```javascript
export default defineConfig({
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'es', 'fr', 'de'],

    routing: {
      prefixDefaultLocale: false,  // /about vs /en/about
      redirectToDefaultLocale: false, // default since Astro 6; only valid with prefixDefaultLocale: true
    },

    fallback: {
      es: 'en',  // Fallback to English for Spanish
    },
  },
});
```

## Image Configuration

```javascript
export default defineConfig({
  image: {
    // Allowed remote image domains
    domains: ['example.com', 'cdn.example.com'],

    // Allowed remote patterns
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.amazonaws.com',
      },
    ],

    // Image service configuration
    service: {
      entrypoint: 'astro/assets/services/sharp',
      config: {
        limitInputPixels: false,
      },
    },
  },
});
```

## Redirects

```javascript
export default defineConfig({
  redirects: {
    '/old-page': '/new-page',
    '/old-blog/[...slug]': '/blog/[...slug]',
    '/twitter': {
      status: 302,
      destination: 'https://twitter.com/astrodotbuild',
    },
  },
});
```

## TypeScript Configuration

### tsconfig.json

```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@layouts/*": ["src/layouts/*"]
    }
  }
}
```

Available presets:
- `astro/tsconfigs/base` - Minimal
- `astro/tsconfigs/strict` - Recommended
- `astro/tsconfigs/strictest` - Maximum strictness

### Type Declarations

```typescript
// src/env.d.ts
/// <reference types="astro/client" />

interface ImportMetaEnv {
  readonly PUBLIC_API_URL: string;
  readonly DATABASE_URL: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

declare namespace App {
  interface Locals {
    user: {
      id: string;
      name: string;
    } | null;
  }
}
```

## Environment Variables

### .env Files

```bash
# .env
PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgresql://localhost/mydb
SECRET_KEY=abc123
```

### Access in Code

```astro
---
// Server-side (all variables)
const dbUrl = import.meta.env.DATABASE_URL;

// Client-side (only PUBLIC_ prefixed)
const apiUrl = import.meta.env.PUBLIC_API_URL;

// Built-in variables
const mode = import.meta.env.MODE;      // 'development' | 'production'
const prod = import.meta.env.PROD;      // boolean
const dev = import.meta.env.DEV;        // boolean
const site = import.meta.env.SITE;      // From astro.config site
const base = import.meta.env.BASE_URL;  // From astro.config base
---
```

### Type-Safe Environment (Astro 5+)

```javascript
// astro.config.mjs
import { defineConfig, envField } from 'astro/config';

export default defineConfig({
  env: {
    schema: {
      PUBLIC_API_URL: envField.string({
        context: 'client',
        access: 'public',
        default: 'https://api.example.com',
      }),
      DATABASE_URL: envField.string({
        context: 'server',
        access: 'secret',
      }),
      PORT: envField.number({
        context: 'server',
        access: 'public',
        default: 4321,
      }),
      FEATURE_FLAG: envField.boolean({
        context: 'client',
        access: 'public',
        default: false,
      }),
    },
  },
});
```

```astro
---
import { PUBLIC_API_URL, DATABASE_URL } from 'astro:env/server';
import { PUBLIC_API_URL } from 'astro:env/client';
---
```

## Session Configuration (Astro 5.7+)

```javascript
import { defineConfig, sessionDrivers } from 'astro/config';

export default defineConfig({
  session: {
    driver: sessionDrivers.redis({
      url: process.env.REDIS_URL,
    }),
    // Or use filesystem, memory, etc.
  },
});
```

## Experimental Features

Flags change every minor — check the docs before adding any. **Remove** these if you find them; they are stable/default now:

| Old flag | Now |
|---|---|
| `experimental.csp` | `security: { csp: true }` (Astro 6) |
| `experimental.fonts` | top-level `fonts` (Astro 6) |
| `experimental.liveContentCollections` | default (Astro 6) |
| `experimental.preserveScriptOrder`, `staticImportMetaEnv`, `headingIdCompat` | default (Astro 6) |
| `experimental.failOnPrerenderConflict` | `prerenderConflictBehavior` (Astro 6) |
| `experimental.rustCompiler`, `queuedRendering`, `advancedRouting` | default (Astro 7) |
| `experimental.cache`, `experimental.routeRules`, `experimental.logger` | top-level `cache`, `routeRules`, `logger` (Astro 7) |

Server islands (`server:defer`) have been stable since Astro 5.

## Full Example

```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import tailwindcss from '@tailwindcss/vite';
import mdx from '@astrojs/mdx';
import sitemap from '@astrojs/sitemap';
import vercel from '@astrojs/vercel';

export default defineConfig({
  site: 'https://mysite.com',

  adapter: vercel(), // static by default; `prerender = false` per route

  integrations: [
    react(),
    mdx(),
    sitemap(),
  ],

  prefetch: {
    prefetchAll: true,
  },

  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'es'],
  },

  image: {
    domains: ['images.unsplash.com'],
  },

  markdown: {
    syntaxHighlight: 'shiki',
    shikiConfig: {
      theme: 'github-dark',
    },
  },

  vite: {
    plugins: [tailwindcss()], // Tailwind v4 (the @astrojs/tailwind integration is deprecated)
    resolve: {
      alias: {
        '@': '/src',
      },
    },
  },
});
```

## Best Practices

1. **Start with strict TypeScript** - Better type safety
2. **Use path aliases** - Cleaner imports
3. **Configure image domains** - Security whitelist
4. **Set site URL** - Required for sitemaps and canonical URLs
5. **Stay static, opt routes into SSR** - `prerender = false` only where needed
6. **Enable prefetching** - Faster navigation
7. **Configure Markdown plugins** - Enhanced content
8. **Type your env variables** - Catch errors early
