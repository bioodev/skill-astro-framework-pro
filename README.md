# astro-framework-pro

An agent skill for building and reviewing **Astro 5–7** sites (verified against **Astro 7.3.5**), with guidance on when to use each rendering and hydration strategy, not just the syntax.

Most Astro skills are stuck on Astro 4/5 APIs (`output: 'hybrid'`, `entry.render()`, `Astro.glob()`, Astro DB). This one checks the installed version first and uses the APIs that are current today: Content Layer and live collections, server islands, route caching (`Astro.cache`, `routeRules`), `src/fetch.ts`, sessions, `astro:env`, the Sätteri Markdown processor and Cloudflare adapter v13+.

## What it covers

- **Decision frameworks:** static vs on-demand rendering, static vs server island vs client island, which `client:*` directive to use, which content loader, when to cache
- **Critical rules and common mistakes:** removed and deprecated APIs across v5 → v6 → v7, and the problems they cause
- **Essential API summary:** components, collections, routing, endpoints, actions, middleware, sessions, env, images, i18n, view transitions and config, all in `SKILL.md`
- **18 on-demand references** in `references/`. Each covers one topic (actions, adapters, caching, images, migration, etc.) and is loaded only when the task needs it
- **Migration guide:** symptom → cause → fix table for upgrading to Astro 6 and 7
- **Accessibility and review checklist** to run before finishing any task

## Installation

### Claude Code (manual)

```bash
git clone <this-repo> ~/skills/astro-framework-pro
ln -s ~/skills/astro-framework-pro ~/.claude/skills/astro-framework-pro
```

Or copy the folder into `~/.claude/skills/` (user-wide) or `.claude/skills/` (per project).

### Other agents

The skill follows the standard `SKILL.md` format and works with any agent that supports skills (Cursor, Codex, Gemini CLI, etc.). Place the folder wherever your agent loads skills from.

## Usage

The skill activates automatically when you work with `.astro` files, `astro.config.*` or `content.config.ts`, or mention Astro. You can also invoke it explicitly:

```
/astro-framework-pro add a blog with content collections and on-demand search
```

Example tasks it's built for:

- "Set up a blog with collections, tags and RSS"
- "Make the product page static but show live price and stock"
- "Deploy this to Cloudflare and add sessions"
- "Upgrade this project from Astro 5 to 7"
- "Review this Astro project for performance and accessibility issues"

## Structure

```
astro-framework-pro/
├── SKILL.md          # version check, decision frameworks, rules, API summary
├── references/       # 18 topic files, loaded on demand
└── LICENSE
```

## Optional external sources

The skill is self-contained. When the installed Astro version is newer than 7.3, or something isn't covered, it points the agent to:

- **context7:** `/withastro/docs`, for current API usage
- **Astro Docs MCP:** `https://mcp.docs.astro.build/mcp`
- **DeepWiki:** `withastro/astro`, for internals only (its index can lag behind releases)

For SEO work (JSON-LD, sitemaps, OG images, IndexNow), pair it with [`jdevalk/astro-seo`](https://github.com/jdevalk/skills).

## Maintenance

Astro ships a major version every few months. The `metadata.last-verified` field in `SKILL.md` records when the content was last checked against the official docs. Re-verify after each Astro major, starting with `references/migration.md`.

## Credits

Built on and heavily revised from:

- [delineas/astro-framework-agents](https://github.com/delineas/astro-framework-agents) (MIT): base references and decision trees
- [incluud/astro-agent-skills](https://github.com/incluud/astro-agent-skills) (MIT): accessibility and review guidance
- [oakoss/agent-skills](https://github.com/oakoss/agent-skills) (MIT): common-mistakes table and shared island state

All content was re-verified against the official [Astro docs](https://docs.astro.build). This process fixed removed APIs, invented packages and incorrect signatures found in the originals.

## License

MIT © bioodev. See [LICENSE](LICENSE).
