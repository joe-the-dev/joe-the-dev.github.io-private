# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```
npm start              # dev server (Vite) on port 3000
npm run build           # tsc typecheck + vite build -> ./build
npm test                 # vitest run (single run, CI mode)
npm run test:watch       # vitest watch mode
npm run lint              # eslint src
npm run format             # prettier --write src
npm run format:check        # prettier --check src
npm run deploy                # gh-pages -d build (manual deploy; normally CI does this via develop push)
```

Run a single test file: `npx vitest run src/App.test.tsx`. Run tests matching a name: `npx vitest run -t "pattern"`.

CI (`.github/workflows/pr-checks.yml`) runs lint, format:check, and test+build on every PR into `develop`. `deploy.yml` builds and pushes `build/` to the external `vuthanhdatdev/vuthanhdatdev.github.io` repo's `gh-pages` branch on every push to `develop`.

## Architecture

This is a static personal site (Vite + React 18 + TypeScript) with two independent sections sharing one app shell:

1. **Portfolio (`/`)** — `src/pages/portfolio.page.tsx`, driven by `src/App.tsx`. Portfolio content is *not* local data: `usePortfolioData` (`src/hooks/usePortfolioData.ts`) fetches JSON from `environment.portfolioDataUrl` (an external GitHub-hosted `data.json`, configurable via `VITE_PORTFOLIO_DATA_URL`) and caches it in `sessionStorage` under `portfolioData`. `src/data/portfolio.data.ts` only defines the `PortfolioData` TypeScript shape, not the content itself. Section components (`about`, `award`, `education`, `experience`, `interests`, `skill`) live in `src/components/*.component.tsx` and use `@makotot/ghostui`'s `Scrollspy` for scroll-linked nav highlighting.

2. **Blog (`/blog/*`)** — routed with `@tanstack/react-router` (route tree assembled in `src/router.ts` from files under `src/routes/`, using the file-based `createRoute`/`getParentRoute` pattern, not TanStack's file-route codegen). Blog posts are **not stored in this repo or a database** — they are markdown files with frontmatter (`title`, `date`, `description`, `tags`, `draft`) in a *separate* GitHub repo, read/written directly through the GitHub Contents API (`src/lib/github-api.ts`) using `VITE_GITHUB_OWNER`/`VITE_GITHUB_REPO`/`VITE_GITHUB_POSTS_PATH`. Auth is GitHub OAuth via Supabase (`src/auth/useAuth.ts`, `src/lib/supabase.ts`); the Supabase session's `provider_token` *is* the GitHub token used directly against the GitHub API for create/update/delete/draft-toggle — there is no separate backend. Draft posts are filtered client-side in `fetchPosts` (visible only when a `githubToken` is present).

3. **`src/lib/environment.ts`** is the single source of truth for all runtime config (`import.meta.env.VITE_*`), with fallback defaults. Add new env vars here rather than reading `import.meta.env` elsewhere, and wire them into both `.env.example` and the two GitHub Actions workflows if they're needed at build time.

Routing note: `src/router.ts` also handles the GitHub Pages SPA 404 redirect trick (reading a `redirect` query param and replacing history) since this is a static host with no server-side routing.
