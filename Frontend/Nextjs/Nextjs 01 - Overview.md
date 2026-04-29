# 01 — Overview

🔑 Next.js is a full-stack React framework: file-system routing, server rendering, data fetching, bundling, and deployment in one toolchain.

Source: https://nextjs.org/docs

## Two Routers, One Repo
| Router | Status | Lives in |
|---|---|---|
| **App Router** | Default since 13.4, primary in 16 | `app/` |
| **Pages Router** | Legacy, still supported | `pages/` |

App Router is built on **React Server Components**; Pages Router is the older `getServerSideProps` / `getStaticProps` model. New work goes in `app/`.

## What You Get
- **RSC + Server Actions** — server-first by default, opt into client with `"use client"`.
- **Cache Components** (v16 GA) — `"use cache"` directive replaces experimental_ppr.
- **File-system routing** — folders are URL segments; `page.tsx` exposes a route.
- **Built-in optimization** — `next/image`, `next/font`, automatic code splitting, prefetch.
- **First-class TypeScript** — typed routes, typed `params`/`searchParams` as Promises.

## Build Targets
| Target | Use case |
|---|---|
| **Node.js server** | `next start` — full feature set, default |
| **Edge runtime** | Per-route opt-in for low-latency, restricted APIs |
| **Static export** | `output: 'export'` — HTML/CSS/JS only, no server features |
| **Standalone** | `output: 'standalone'` — minimal Docker image with traced deps |

## Project Skeleton
```
app/
  layout.tsx     # root <html>/<body>
  page.tsx       # / route
  globals.css
public/          # static assets, served from /
next.config.ts
tsconfig.json
```

⚠️ The legacy `middleware.ts` was renamed to `proxy.ts` in v16 — see [[Middleware]] note for migration.

💡 `src/` is optional; if present, `app/` lives inside it.

## Tags
[[Nextjs]] [[App Router]] [[RSC]] [[Vercel]] [[React]]
