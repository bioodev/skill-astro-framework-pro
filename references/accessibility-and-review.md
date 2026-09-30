# Accessibility, Project Shape & Review Checklist

Load before finalizing any Astro work or when asked to review an Astro project.

## Project Shape

```text
src/
  components/   layouts/   pages/   styles/   assets/
  content/      # collection source files
  actions/      # defineAction (index.ts)
  middleware.ts
  content.config.ts   live.config.ts
public/         # served as-is, never processed
```

Follow the existing project's conventions over this template. Don't invent novel folder structures without a project-specific reason.

## Accessibility Baseline

Astro ships mostly static HTML, so accessibility is mostly about the markup you write:

- Semantic landmarks: `header`, `nav`, `main` (one per page), `article`, `aside`, `footer`
- One `h1` per page and no skipped heading levels. Layouts must not add a second `h1`.
- `<html lang>` is set, and matches the locale on i18n routes
- Every `<Image>` / `<Picture>` gets meaningful `alt`, or `alt=""` if decorative. Store `coverAlt` next to `image()` in collection schemas.
- Interactive controls are real `<button>` / `<a href>`, reachable by keyboard, with accessible names
- Focus styles are visible. Never apply `outline: none` without a replacement.
- Color contrast meets WCAG AA
- **View transitions:** `<ClientRouter />` announces the new page title to screen readers. `<ClientRouter />` disables all transition animations automatically under `prefers-reduced-motion`. Respect it yourself in any other CSS/JS animation. Test keyboard focus after navigation, and don't break the announcer with custom swap logic.
- **Islands:** hydration must not shift focus or layout. A server-rendered fallback must be usable (or clearly marked as loading) before JS arrives.
- Vanilla `<script>` menus/toggles: keep `aria-expanded` and `aria-controls` in sync, and close on `Escape`

## Performance Baseline

- Give images dimensions (automatic for imported local images) to avoid CLS
- Keep CSS and JS scoped to the component that needs it. Avoid global imports in layouts unless they really are global.
- Prefer simple server-rendered markup over client complexity
- Use Fonts API `<Font cssVariable="..." preload />` only for the primary font

## Review Checklist

Before finishing, verify:

1. **Hydration:** can any `client:*` be removed or downgraded? Is a whole framework component hydrated just for a toggle?
2. **Rendering:** is anything `prerender = false` that could be static? Is anything static that reads cookies or headers?
3. **APIs are current** for the installed version (see migration.md): no `entry.render()`, `Astro.glob`, `'hybrid'`, `z` from `astro:content`, or `Astro.locals.runtime` on Cloudflare v13+
4. **Schemas:** do collections validate everything? Dates use `z.coerce.date()`, booleans have `.default()`, images use `image()`.
5. **Semantics & a11y:** check the baseline above
6. **Serialization:** only serializable props go to `client:*` and `server:defer` components
7. **Env/secrets:** secrets come from `astro:env/server` and never reach client bundles. Nothing depends on boolean `import.meta.env` comparisons.
8. **Build passes:** run `astro check && astro build`. The Astro 7 Rust compiler rejects invalid HTML.
9. **Conventions:** does the code match the existing project's structure and styling system?
