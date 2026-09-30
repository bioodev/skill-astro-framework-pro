# Content Collections

Content collections provide type-safe content management with schema validation using Zod.

## Contents
- Setup (Astro 5+ Content Layer API)
- Built-in Loaders
- Live Collections (Astro 6+, stable)
- Schema Validation with Zod
- Querying Collections
- Rendering Content
- Dynamic Routes with Collections
- Expert Guidance
- Best Practices

## Setup (Astro 5+ Content Layer API)

### Directory Structure

```
src/
├── content.config.ts    # Collection schemas (Astro 5+)
├── content/
│   ├── blog/            # Blog collection
│   │   ├── post-1.md
│   │   ├── post-2.mdx
│   │   └── drafts/      # Subdirectories supported
│   │       └── draft-1.md
│   └── authors/         # Another collection
│       └── john.json
```

> **Note:** The config file is `src/content.config.ts`. Every collection needs a `loader`. The legacy API (`src/content/config.ts`, `type: 'content'|'data'`, `entry.slug`, `entry.render()`, `getEntryBySlug()`) was **removed in Astro 6**.

### Configuration File

```typescript
// src/content.config.ts
import { defineCollection } from 'astro:content';
import { glob, file } from 'astro/loaders';
import { z } from 'astro/zod';

const blogCollection = defineCollection({
  loader: glob({ base: './src/content/blog', pattern: '**/*.{md,mdx}' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    heroImage: z.string().optional(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
    author: z.string(),
  }),
});

const authorsCollection = defineCollection({
  loader: file('src/data/authors.json'),
  schema: z.object({
    name: z.string(),
    email: z.email(),
    bio: z.string(),
    avatar: z.url(),
    social: z.object({
      twitter: z.string().optional(),
      github: z.string().optional(),
    }).optional(),
  }),
});

export const collections = {
  blog: blogCollection,
  authors: authorsCollection,
};
```

## Built-in Loaders

### `glob()` — Multiple files

Loads entries from directories of Markdown, MDX, JSON, YAML, or TOML files:

```typescript
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({
    base: './src/content/blog',
    pattern: '**/*.{md,mdx}',
  }),
  schema: z.object({ title: z.string() }),
});
```

Options: `pattern`, `base`, `generateId()` (custom ID generation), `retainBody` (set `false` to exclude raw body), `deferRender` (set `true` on large Markdown collections to render at page generation instead of during sync, which avoids running out of memory in `astro build`).

### `file()` — Single file

Loads entries from a single JSON, YAML, or TOML file:

```typescript
import { file } from 'astro/loaders';

const authors = defineCollection({
  loader: file('src/data/authors.json'),
  schema: z.object({ name: z.string() }),
});
```

Supports a custom `parser` for non-standard formats (e.g., CSV).

### Custom Loaders

> Loader schemas: an object loader can declare its own schema via `createSchema()` (Astro 6+). The old `schema: async () => ...` function form was removed.

Load from any source (APIs, databases, CMSes):

```typescript
const products = defineCollection({
  loader: async () => {
    const response = await fetch('https://api.example.com/products');
    const data = await response.json();
    return data.map((product: any) => ({
      id: product.id,
      ...product,
    }));
  },
  schema: z.object({ name: z.string(), price: z.number() }),
});
```

### Object Loaders (Advanced)

For full control with incremental updates, caching, and file watching:

```typescript
import type { Loader } from 'astro/loaders';

function myLoader(options: { url: string }): Loader {
  return {
    name: 'my-loader',
    load: async ({ store, meta, logger }) => {
      const lastModified = meta.get('lastModified');
      const data = await fetchData(options.url, lastModified);

      store.clear();
      for (const item of data) {
        store.set({ id: item.id, data: item });
      }

      meta.set('lastModified', new Date().toISOString());
    },
  };
}
```

## Live Collections (Astro 6+, stable)

Live collections fetch data **on every request** (no build-time data store). They live in a separate file, `src/live.config.ts`, and require on-demand rendering (adapter + `prerender = false`).

```typescript
// src/live.config.ts
import { defineLiveCollection } from 'astro:content';
import { z } from 'astro/zod';
import type { LiveLoader } from 'astro/loaders';

function productLoader(config: { apiKey: string }): LiveLoader<Product> {
  return {
    name: 'product-loader',
    loadCollection: async ({ filter }) => {
      const data = await fetchProducts(config.apiKey, filter);
      return { entries: data.map((p) => ({ id: p.sku, data: p })) };
    },
    loadEntry: async ({ filter }) => {
      const product = await fetchProduct(config.apiKey, filter.id);
      if (!product) return undefined;
      return { id: product.sku, data: product };
    },
  };
}

const products = defineLiveCollection({
  loader: productLoader({ apiKey: process.env.STORE_API_KEY! }),
  schema: z.object({ sku: z.string(), name: z.string(), price: z.number() }),
});

export const collections = { products };
```

Query with `getLiveCollection()` / `getLiveEntry()`. They **return errors instead of throwing** — always check `error`:

```astro
---
export const prerender = false;
import { getLiveEntry } from 'astro:content';

const { entry, error, cacheHint } = await getLiveEntry('products', Astro.params.sku!);
if (error || !entry) return Astro.redirect('/404');

// Astro 7 route caching: forward the loader's hint (tags / lastModified)
if (cacheHint) Astro.cache.set(cacheHint);
---
<h1>{entry.data.name}</h1>
```

Error classes: `LiveEntryNotFoundError`, `LiveCollectionValidationError`, `LiveCollectionCacheHintError`, `LiveCollectionError`.

## Schema Validation with Zod

### Common Field Types

```typescript
import { z } from 'astro/zod'; // Zod 4 (Astro 6+)

const schema = z.object({
  // Strings
  title: z.string(),
  slug: z.string().regex(/^[a-z0-9-]+$/),

  // Numbers
  order: z.number().int().positive(),
  rating: z.number().min(0).max(5),

  // Booleans
  featured: z.boolean().default(false),

  // Dates
  pubDate: z.coerce.date(), // Converts strings to Date

  // Arrays
  tags: z.array(z.string()),
  categories: z.array(z.enum(['tech', 'life', 'travel'])),

  // Objects
  author: z.object({
    name: z.string(),
    email: z.email(),
  }),

  // Enums
  status: z.enum(['draft', 'published', 'archived']),

  // Optional fields
  description: z.string().optional(),
  image: z.url().optional(),

  // Default values
  views: z.number().default(0),

  // Unions
  media: z.union([
    z.object({ type: z.literal('image'), src: z.string() }),
    z.object({ type: z.literal('video'), url: z.string() }),
  ]),
});
```

### Image Schema

```typescript
// Astro 5+ with loader
import { defineCollection } from 'astro:content';
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';

const blog = defineCollection({
  loader: glob({ base: './src/content/blog', pattern: '**/*.md' }),
  schema: ({ image }) => z.object({
    title: z.string(),
    cover: image(), // Validates and optimizes images
    coverAlt: z.string(),
  }),
});
```

### Reference Other Collections

```typescript
import { defineCollection, reference } from 'astro:content';
import { z } from 'astro/zod';

const blog = defineCollection({
  loader: glob({ base: './src/content/blog', pattern: '**/*.md' }),
  schema: z.object({
    title: z.string(),
    author: reference('authors'), // References authors collection
    relatedPosts: z.array(reference('blog')).optional(),
  }),
});
```

## Querying Collections

### getCollection()

```astro
---
import { getCollection } from 'astro:content';

// Get all entries
const allPosts = await getCollection('blog');

// Filter entries
const publishedPosts = await getCollection('blog', ({ data }) => {
  return data.draft !== true;
});

// Filter by date
const recentPosts = await getCollection('blog', ({ data }) => {
  return data.pubDate > new Date('2024-01-01');
});

// Sort entries
const sortedPosts = (await getCollection('blog'))
  .sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf());
---

<ul>
  {sortedPosts.map((post) => (
    <li>
      <a href={`/blog/${post.id}`}>{post.data.title}</a>
      <time>{post.data.pubDate.toLocaleDateString()}</time>
    </li>
  ))}
</ul>
```

### getEntry()

```astro
---
import { getEntry } from 'astro:content';

// Get single entry by collection and id
const post = await getEntry('blog', 'my-first-post');

// Get single entry by reference
const author = await getEntry(post.data.author);

// Check if entry exists
if (!post) {
  return Astro.redirect('/404');
}
---

<article>
  <h1>{post.data.title}</h1>
  <p>By {author.data.name}</p>
</article>
```

### getEntries()

```astro
---
import { getEntry, getEntries } from 'astro:content';

const post = await getEntry('blog', 'my-post');

// Get multiple referenced entries
const relatedPosts = await getEntries(post.data.relatedPosts);
---
```

## Rendering Content

### `render()` from `astro:content`

```astro
---
import { getEntry, render } from 'astro:content';

const post = await getEntry('blog', 'my-post');
const { Content, headings } = await render(post);
---

<article>
  <h1>{post.data.title}</h1>

  <!-- Table of contents from headings -->
  <nav>
    {headings.map((h) => (
      <a href={`#${h.slug}`} style={`margin-left: ${h.depth * 10}px`}>
        {h.text}
      </a>
    ))}
  </nav>

  <!-- Rendered content -->
  <Content />
</article>
```

## Dynamic Routes with Collections

```astro
---
// src/pages/blog/[...slug].astro
import { getCollection, render } from 'astro:content';

export async function getStaticPaths() {
  const posts = await getCollection('blog');
  return posts.map((post) => ({
    params: { slug: post.id },
    props: { post },
  }));
}

const { post } = Astro.props;
const { Content } = await render(post);
---

<article>
  <h1>{post.data.title}</h1>
  <Content />
</article>
```

> **Note:** Use `post.id` for routing params (`entry.slug` no longer exists). Use `post.filePath` if you need the source file.

## Expert Guidance

### Loader Selection Decision Tree

```
Local markdown/MDX files → glob() loader
Single JSON/YAML data file → file() loader
Remote API/CMS data at build → Custom async loader function
Remote data fresh per-request → Live Loader (Astro 6+)
```

### Performance: `retainBody` for Large Sites

For sites with >1000 content entries where you only need frontmatter data (e.g., listing pages, tag indexes), disable body storage:

```typescript
const blog = defineCollection({
  loader: glob({
    base: './src/content/blog',
    pattern: '**/*.md',
    retainBody: false, // Significantly reduces data store size
  }),
  schema: z.object({ title: z.string(), pubDate: z.coerce.date() }),
});
```

Only use `retainBody: false` for collections where you don't call `render()`. If you need to render content on detail pages, keep the default (`true`).

### Migration Pitfalls (legacy → Astro 6/7)

| What changed | Old (removed in Astro 6) | Current |
|---|---|---|
| Config file | `src/content/config.ts` | `src/content.config.ts` (+ `src/live.config.ts` for live) |
| Collection type | `type: 'content'` / `type: 'data'` | `loader: glob(...)` / `loader: file(...)` |
| Zod import | `import { z } from 'astro:content'` / `'astro:schema'` (deprecated) | `import { z } from 'astro/zod'` |
| Zod formats (Zod 4) | `z.string().email()` / `.url()` / `.uuid()` | `z.email()` / `z.url()` / `z.uuid()` |
| Zod error msg | `{ message: '...' }` | `{ error: '...' }` |
| Rendering | `entry.render()` | `import { render } from 'astro:content'; await render(entry)` |
| Route param | `entry.slug` | `entry.id` |
| Lookup | `getEntryBySlug()` / `getDataEntryById()` | `getEntry(collection, id)` |
| Temporary escape hatch | — | `legacy: { collectionsBackwardsCompat: true }` (migrate ASAP) |

**Zod 4 gotcha:** `.default()` after `.transform()` now takes the **output** type (`z.string().transform(Number).default(0)`); use `.prefault()` for the old behavior.

### Schema Design Anti-Patterns

- **`z.date()` for frontmatter dates** → Always use `z.coerce.date()`. YAML/frontmatter dates arrive as strings; `z.date()` rejects them silently
- **Missing `.default()` on optional booleans** → `draft: z.boolean().optional()` means `undefined`, not `false`. Use `z.boolean().default(false)` so filtering logic works correctly
- **String paths for images** → Use the `image()` schema helper to enable Astro's image optimization pipeline. String URLs bypass optimization entirely

## Best Practices

1. **Always define schemas** - Type safety and validation
2. **Use `z.coerce.date()`** - Handles string dates from frontmatter
3. **Set sensible defaults** - `.default()` reduces required fields
4. **Filter drafts in production** - Don't expose unpublished content
5. **Use references for relationships** - Type-safe cross-collection links
6. **Keep slugs URL-friendly** - Validate with regex
7. **Use image schema for images** - Enables optimization
