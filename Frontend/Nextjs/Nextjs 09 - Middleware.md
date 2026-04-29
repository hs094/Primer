# 09 — Middleware (now `proxy.ts` in v16)

🔑 A single root-level file runs server code **before** any route renders — ideal for auth gates, locale rewrites, headers, A/B cookies. Defaults to the Edge runtime for low-latency.

Source: https://nextjs.org/docs/app/api-reference/file-conventions/proxy

⚠️ **Renamed in Next.js 16**: `middleware.ts` → `proxy.ts`, function `middleware` → `proxy`. Migration codemod: `npx @next/codemod@canary middleware-to-proxy .`. The behavior is identical; the rename is to avoid Express.js confusion.

## Shape
```ts
// proxy.ts (or middleware.ts on <16)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  const session = request.cookies.get('session')
  if (!session && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
}
```

## `NextResponse` Toolkit
| Method | Purpose |
|---|---|
| `.next()` | Continue to the route, optionally with mutated request headers |
| `.redirect(url)` | 307/308 redirect |
| `.rewrite(url)` | Serve a different route without changing the visible URL |
| `.json(body)` | Respond directly (e.g. 401 for an API gate) |
| `.cookies.set/get/delete` | Cookie ops on request or response |

## `matcher` Patterns
```ts
export const config = {
  matcher: [
    // Skip Next internals + static
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
    { source: '/api/:path*', has: [{ type: 'header', key: 'authorization' }] },
  ],
}
```
`matcher` values must be **static** — they're analyzed at build time.

⚠️ Proxy runs on every request including Server Action POSTs. Don't rely on it as the *only* auth check — re-validate inside [[Server Actions]] / Route Handlers.

💡 Common uses: auth gating, geo/locale rewrites, feature flags via cookie, request ID injection, CORS preflight handling.

## Tags
[[Nextjs]] [[App Router]] [[Edge]] [[Auth]]
