# Images

Astro provides built-in image optimization through the `astro:assets` module.

## Contents
- Image Component
- Picture Component
- Responsive Images: `layout` (5.10+)
- SVG Components (5.7+)
- Above-the-fold images: `priority`
- getImage Function
- Image Paths
- Content Collections Images
- Image Formats
- Quality Settings
- Responsive Images
- Configuration
- Markdown Images
- Background Images
- Best Practices

## Image Component

### Basic Usage

```astro
---
import { Image } from 'astro:assets';
import heroImage from '../images/hero.jpg';
---

<Image src={heroImage} alt="Hero image" />
```

Output: Optimized image with proper format, dimensions, and lazy loading.

### With Properties

```astro
---
import { Image } from 'astro:assets';
import photo from '../images/photo.jpg';
---

<Image
  src={photo}
  alt="A description"
  width={800}
  height={600}
  format="webp"
  quality={80}
  loading="lazy"
  decoding="async"
  class="rounded-lg shadow-md"
/>
```

### Remote Images

```astro
---
import { Image } from 'astro:assets';
---

<Image
  src="https://example.com/image.jpg"
  alt="Remote image"
  width={400}
  height={300}
  inferSize={false}
/>
```

For remote images, `width` and `height` are required (or use `inferSize`).

### Infer Size for Remote Images

```astro
<Image
  src="https://example.com/image.jpg"
  alt="Remote image"
  inferSize
/>
```

## Picture Component

Provides responsive images with multiple formats:

```astro
---
import { Picture } from 'astro:assets';
import photo from '../images/photo.jpg';
---

<Picture
  src={photo}
  alt="Responsive image"
  formats={['avif', 'webp', 'jpg']}
  widths={[400, 800, 1200]}
  sizes="(max-width: 600px) 400px, (max-width: 1000px) 800px, 1200px"
/>
```

Output:

```html
<picture>
  <source srcset="..." type="image/avif" sizes="...">
  <source srcset="..." type="image/webp" sizes="...">
  <img src="..." alt="Responsive image" loading="lazy" decoding="async">
</picture>
```

## Responsive Images: `layout` (5.10+)

Let Astro generate `srcset`/`sizes` instead of writing them by hand:

```astro
<Image src={hero} alt="..." layout="constrained" width={1200} height={600} />
<Image src={banner} alt="..." layout="full-width" />
<Image src={icon} alt="..." layout="fixed" width={64} height={64} />
<Image src={photo} alt="..." layout="constrained" fit="contain" position="top" />
```

- `constrained`: scales down to the container, never above the given size (the usual choice)
- `full-width`: always fills the container width
- `fixed`: keeps its dimensions (with density variants)

Set a site-wide default and the base styles in config:

```javascript
image: {
  layout: 'constrained',
  responsiveStyles: true,   // leave false with Tailwind 4 if you want Tailwind classes to win
  // objectFit: 'cover', objectPosition: 'center',
}
```

Images in `public/` are never optimized or responsive. Each responsive image produces several files: this costs build time on prerendered pages, or transformations (and possibly money) at request time on on-demand pages.

**Astro 6 defaults:** giving both `width` and `height` with a different aspect ratio **crops** (`fit` defaults to `cover`), and images are **never upscaled**.

## SVG Components (5.7+)

```astro
---
import Logo from '../assets/logo.svg';   // default import is an Astro component (inlined SVG)
---
<Logo width={48} height={48} fill="currentColor" aria-label="Acme" />
```

Props override attributes in the file. Like any Astro component, SVG components can't be used inside framework components directly; pass them in as children from the `.astro` side.

## Above-the-fold images: `priority`

For the LCP image (hero), add `priority` to `<Image />` / `<Picture />`. It sets `loading="eager"`, `decoding="sync"` and `fetchpriority="high"`. Use it on one or two images per page at most.

```astro
<Image src={hero} priority alt="..." />
```

## getImage Function

For programmatic image processing:

```astro
---
import { getImage } from 'astro:assets';
import background from '../images/background.jpg';

const optimizedBg = await getImage({
  src: background,
  format: 'webp',
  width: 1920,
  quality: 80,
});
---

<div style={`background-image: url(${optimizedBg.src})`}>
  Content with background
</div>
```

## Image Paths

### Local Images (src/)

```astro
---
// Import from src directory
import hero from '../images/hero.jpg';
import logo from '@/assets/logo.png'; // Using alias
---

<Image src={hero} alt="Hero" />
<Image src={logo} alt="Logo" />
```

### Public Folder Images

Images in `public/` are not optimized:

```astro
<!-- Not optimized, served as-is -->
<img src="/images/static-image.jpg" alt="Static" />

<!-- Can still use Image component with remote-like syntax -->
<Image
  src="/images/static-image.jpg"
  alt="Static"
  width={400}
  height={300}
/>
```

### Dynamic Imports

```astro
---
const images = import.meta.glob<{ default: ImageMetadata }>(
  '../images/*.{jpg,png,gif}'
);

const imagePaths = Object.keys(images);
---

{imagePaths.map(async (path) => {
  const image = await images[path]();
  return <Image src={image.default} alt="" />;
})}
```

## Content Collections Images

### Schema with Image

```typescript
// src/content.config.ts
import { defineCollection } from 'astro:content';
import { glob } from 'astro/loaders';
import { z } from 'astro/zod';

const blog = defineCollection({
  loader: glob({ base: './src/content/blog', pattern: '**/*.{md,mdx}' }),
  schema: ({ image }) => z.object({
    title: z.string(),
    cover: image(),
    coverAlt: z.string(),
  }),
});

export const collections = { blog };
```

### Markdown Frontmatter

```markdown
---
title: "My Post"
cover: "./images/cover.jpg"
coverAlt: "Post cover image"
---

Content here...
```

### Using in Template

```astro
---
import { Image } from 'astro:assets';
import { getEntry } from 'astro:content';

const post = await getEntry('blog', 'my-post');
---

<Image src={post.data.cover} alt={post.data.coverAlt} />
```

## Image Formats

Supported output formats:
- `webp` (default for optimization)
- `avif` (best compression, slower)
- `png` (lossless)
- `jpg`/`jpeg` (lossy)
- `svg`: kept as SVG by default. Since Astro 6 an SVG source **can** be rasterized by passing `format="png"`/`"webp"`.
- `gif`: animated GIFs fall back to `.gif`

```astro
<Image src={photo} alt="Photo" format="avif" quality={60} />
```

## Quality Settings

```astro
<!-- Lower quality, smaller file -->
<Image src={photo} alt="" quality="low" />
<Image src={photo} alt="" quality={50} />

<!-- Medium quality (default) -->
<Image src={photo} alt="" quality="mid" />
<Image src={photo} alt="" quality={75} />

<!-- Higher quality -->
<Image src={photo} alt="" quality="high" />
<Image src={photo} alt="" quality={90} />

<!-- Maximum quality -->
<Image src={photo} alt="" quality="max" />
<Image src={photo} alt="" quality={100} />
```

## Responsive Images

### With Picture

```astro
<Picture
  src={hero}
  alt="Hero"
  widths={[640, 768, 1024, 1280, 1536]}
  sizes="(max-width: 640px) 640px, (max-width: 768px) 768px, (max-width: 1024px) 1024px, (max-width: 1280px) 1280px, 1536px"
  formats={['avif', 'webp', 'jpg']}
/>
```

### With densities

```astro
<Image
  src={logo}
  alt="Logo"
  width={200}
  densities={[1, 2, 3]}
/>

<!-- Output srcset with 1x, 2x, 3x versions -->
```

## Configuration

### Image Service

```javascript
// astro.config.mjs
export default defineConfig({
  image: {
    // Sharp is default, or use custom service
    service: {
      entrypoint: 'astro/assets/services/sharp',
      config: {
        limitInputPixels: false,
      },
    },

    // Allowed remote domains
    domains: ['example.com', 'cdn.example.com'],

    // Allowed remote patterns
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.amazonaws.com',
      },
    ],
  },
});
```

### No-op / External Services

Skip Sharp entirely (e.g. on hosts where it can't install):

```javascript
import { defineConfig, passthroughImageService } from 'astro/config';
export default defineConfig({ image: { service: passthroughImageService() } });
```

Hosted image CDNs usually ship their **own components** rather than an image service. Cloudinary, for example, uses `npm i astro-cloudinary` + `import { CldImage } from 'astro-cloudinary'`. On Cloudflare, the adapter's `imageService` option controls this (default `'cloudflare-binding'`).

## Markdown Images

### Using Relative Paths

```markdown
<!-- src/content/blog/post.md -->
![Alt text](./images/image.jpg)
```

### Using Image Component in MDX

```mdx
---
title: My Post
---

import { Image } from 'astro:assets';
import photo from './images/photo.jpg';

# My Post

<Image src={photo} alt="Photo" width={600} />

Regular markdown images still work:
![Alt](./images/other.jpg)
```

## Background Images

```astro
---
import { getImage } from 'astro:assets';
import bg from '../images/background.jpg';

const optimizedBg = await getImage({ src: bg, format: 'webp' });
---

<section class="hero" style={`background-image: url(${optimizedBg.src})`}>
  <h1>Welcome</h1>
</section>

<style>
  .hero {
    background-size: cover;
    background-position: center;
    min-height: 100vh;
  }
</style>
```

## Best Practices

1. **Always use Image component for local images** - Automatic optimization
2. **Provide alt text** - Accessibility requirement
3. **Use Picture for hero images** - Multiple formats and sizes
4. **Specify width/height for CLS** - Prevents layout shift
5. **Use WebP/AVIF formats** - Better compression
6. **Configure remote domains** - Security whitelist
7. **Use quality settings appropriately** - Balance size vs quality
8. **Import images, don't use strings** - Enables optimization
9. **Use content collection image schema** - Type-safe image handling
