# Next.js — INDEX

🔑 A 12-note knowledge pack on Next.js 16 — App Router, RSC, Cache Components, Server Actions, deployment. TypeScript throughout.

Source: https://nextjs.org/docs

## Notes
1. [[Nextjs 01 - Overview]] — full-stack React framework, App vs Pages router, build targets.
2. [[Nextjs 02 - App Router]] — `app/` file conventions, dynamic + catch-all routes, route groups.
3. [[Nextjs 03 - React Server Components]] — server-default rendering, `"use client"` boundary, prop serialization.
4. [[Nextjs 04 - Server Actions]] — `"use server"`, form actions, mutations, revalidation.
5. [[Nextjs 05 - Cache Components]] — `"use cache"`, `cacheLife` / `cacheTag`, replaces experimental_ppr.
6. [[Nextjs 06 - Streaming and Suspense]] — `loading.tsx`, `<Suspense>`, progressive HTML.
7. [[Nextjs 07 - Data Fetching]] — `fetch` extensions, parallel vs sequential, `generateStaticParams`.
8. [[Nextjs 08 - Parallel and Intercepting Routes]] — `@slot` folders, `(.)` modal pattern.
9. [[Nextjs 09 - Middleware]] — `middleware.ts` (renamed `proxy.ts` in v16), matcher, `NextResponse`.
10. [[Nextjs 10 - Image and Font]] — `next/image`, `next/font`, zero CLS.
11. [[Nextjs 11 - Metadata and SEO]] — `generateMetadata`, OG images, sitemap, robots.
12. [[Nextjs 12 - Deployment]] — Vercel, Node, Docker standalone, OpenNext, ISR.

## Mental Model
Server-first by default. Mark interactive leaves with `"use client"`, cacheable units with `"use cache"`, mutations with `"use server"`. Stream slow chunks behind `<Suspense>`. The router lives in the file system.

## Tags
[[Nextjs]] [[App Router]] [[RSC]] [[Server Actions]] [[Cache Components]] [[Streaming]] [[Vercel]]
