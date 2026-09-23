# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm run dev` — start the dev server (Turbopack) at http://localhost:3000
- `npm run build` — production build (Turbopack)
- `npm run start` — run the production build
- `npm run lint` — run ESLint

There is no test runner configured yet.

## Architecture

Next.js 15 App Router project, TypeScript, Tailwind CSS v4 (via `@tailwindcss/postcss`), no `src/` directory — routes live directly under `app/`.

Folder layout:

- `app/` — routes, layouts, and route-level UI (App Router convention: `page.tsx`, `layout.tsx`, etc.)
- `components/` — shared, reusable UI components used across routes
- `lib/` — framework-agnostic helpers, data access, and business logic
- `types/` — shared TypeScript types/interfaces used across the app
- `public/` — static assets served as-is

Path alias `@/*` maps to the project root (see `tsconfig.json`), so imports use `@/components/...`, `@/lib/...`, `@/types/...` rather than relative paths across folders.

`components/`, `lib/`, and `types/` are currently empty skeletons (placeholder `.gitkeep` files) — populate them as real code is added.
