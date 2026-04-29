# 05 — Cache Components

🔑 The `"use cache"` directive is Next.js 16's GA replacement for experimental Partial Prerendering (PPR) — mark a file, component, or function as cacheable; the rest of the route stays dynamic and streams in.

Source: https://nextjs.org/docs/app/api-reference/directives/use-cache

## Enable
```ts
// next.config.ts
import type { NextConfig } from 'next'
const config: NextConfig = { cacheComponents: true }
export default config
```

## Three Scopes
```tsx
// File-level — every export is cached
'use cache'
export default async function Page() { /* ... */ }

// Component-level
export async function ProductGrid() {
  'use cache'
  return <ul>{/* ... */}</ul>
}

// Function-level
export async function getPosts() {
  'use cache'
  return fetch('https://api.example.com/posts').then(r => r.json())
}
```

## Cache Key
Auto-derived from: build ID + function location hash + serialized args + closed-over vars. Different args → different entry.

## Tuning Lifetime + Invalidation
```tsx
import { cacheLife, cacheTag } from 'next/cache'

export async function getProducts() {
  'use cache'
  cacheLife('hours')      // built-in profiles: seconds, minutes, hours, days, weeks, max
  cacheTag('products')    // invalidate later via revalidateTag('products')
  return db.products.findMany()
}
```

Defaults: client stale 5 min, server revalidate 15 min, never expires by time.

## Constraints
| Allowed | Forbidden inside `use cache` |
|---|---|
| Pure args, fetch, DB queries | `cookies()`, `headers()`, `searchParams` |
| Pass-through `children` / Server Actions | Class instances, `URL`, raw functions |

⚠️ Read request-time values **outside** the cached scope and pass them as args — otherwise the build hangs with a "Filling a cache during prerender timed out" error.

💡 A route is fully prerendered only when both `layout` and `page` carry `"use cache"`. If only the page is cached, the layout still renders dynamically.

🧪 `NEXT_PRIVATE_DEBUG_CACHE=1 npm run dev` logs every cache hit/miss with a `Cache` prefix.

## Tags
[[Nextjs]] [[Cache Components]] [[RSC]] [[Streaming]]
