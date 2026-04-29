# 05 — Cache Components

🔑 The `"use cache"` directive is Next.js 16's GA replacement for experimental PPR — mark a file, component, or function as cacheable; the rest of the route streams in dynamically.

Source: https://nextjs.org/docs/app/api-reference/directives/use-cache

## Enable
```ts
// next.config.ts
const config = { cacheComponents: true }
export default config
```

## Three Scopes
```tsx
'use cache'                              // file-level
export default async function Page() {}

export async function ProductGrid() {
  'use cache'                            // component-level
  return <ul>{/* ... */}</ul>
}

export async function getPosts() {
  'use cache'                            // function-level
  return fetch('https://api.example.com/posts').then(r => r.json())
}
```

Cache key = build ID + function-location hash + serialized args + closed-over vars.

## Lifetime + Invalidation
```tsx
import { cacheLife, cacheTag } from 'next/cache'

export async function getProducts() {
  'use cache'
  cacheLife('hours')      // built-in: seconds, minutes, hours, days, weeks, max
  cacheTag('products')    // invalidate: revalidateTag('products')
  return db.products.findMany()
}
```
Defaults: client stale 5 min, server revalidate 15 min, no time expiry.

## Constraints
| OK | Forbidden inside `use cache` |
|---|---|
| Args, fetch, DB queries | `cookies()`, `headers()`, `searchParams` |
| Pass-through `children` / Server Actions | Class instances, `URL`, raw functions |

⚠️ Read request-time values **outside** the cached scope and pass as args, or the build hangs with a "Filling a cache during prerender timed out" error.

💡 A route is fully prerendered only if both `layout` and `page` carry `"use cache"`.

🧪 `NEXT_PRIVATE_DEBUG_CACHE=1 npm run dev` logs every cache hit/miss.

## Tags
[[Nextjs]] [[Cache Components]] [[RSC]] [[Streaming]]
