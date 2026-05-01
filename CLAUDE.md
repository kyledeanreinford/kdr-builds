# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run dev` — Astro dev server with hot reload
- `npm run build` — produces static site in `dist/`
- `npm run preview` — serves the built `dist/` locally

Node >= 22.12 is required (matches the CI and Dockerfile).

There is no test suite, linter, or formatter configured — do not invent commands for them.

## Architecture

This is a static blog built with Astro 6, deployed as a static site served by nginx in a Docker image (see `Dockerfile`, `nginx.conf`). CI in `.github/workflows/build-and-push.yml` builds the site, then on `main` builds and pushes the container to a Harbor registry.

Key pieces:

- **Content collection** — `src/content.config.ts` defines a single `blog` collection that globs `src/content/blog/**/*.{md,mdx}`. The Zod schema is the source of truth for post frontmatter: `title`, `description`, `pubDate` (required), plus optional `updatedDate`, `tags` (array, defaults to `[]`), `draft` (boolean, defaults to `false`), `heroImage`. Adding a new frontmatter field means updating this schema.
- **Routing** — `src/pages/blog/[...slug].astro` generates one page per collection entry via `getStaticPaths`; the slug is the entry `id` (filename without extension). `src/pages/tags/[tag].astro` generates per-tag index pages from the same collection. `src/pages/rss.xml.js` produces the feed.
- **Layout** — all posts render through `src/layouts/BlogPost.astro`, which composes `BaseHead`, `Header`, `Footer`, and `EmailSignup`. There is no per-post layout override; styling lives in component-scoped `<style>` blocks plus `src/styles/global.css`.
- **Site config** — `astro.config.mjs` sets `site: 'https://kyledeanreinford.com'` (used by `@astrojs/sitemap` and the RSS feed). Site title/description constants live in `src/consts.ts`.

## Authoring posts

New posts are markdown/MDX files in `src/content/blog/`. The filename becomes the URL slug. Frontmatter must satisfy the schema in `content.config.ts` — Astro's build will fail loudly on schema violations, which is the intended check.
