# 09 — Middleware (now `proxy.ts` in v16)

🔑 A single root-level file runs server code **before** any route renders — ideal for auth gates, locale rewrites, headers. Defaults to the Edge runtime.

Source: https://nextjs.org/docs/app/api-reference/file-conventions/proxy

⚠️ **Renamed in v16**: `middleware.ts` → `proxy.ts`, function `middleware` → `proxy`. Codemod: `npx @next/codemod@canary middleware-to-proxy .`. Behavior is unchanged.

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
| `.next()` | Continue, optionally with mutated request headers |
| `.redirect(url)` | 307/308 redirect |
| `.rewrite(url)` | Different route, same visible URL |
| `.json(body)` | Respond directly (e.g. 401 API gate) |
| `.cookies.set/get/delete` | Cookie ops on request or response |

## `matcher`
```ts
export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
    { source: '/api/:path*', has: [{ type: 'header', key: 'authorization' }] },
  ],
}
```
Values must be **static** — analyzed at build time.

⚠️ Proxy runs on Server Action POSTs too. Don't make it the only auth check — re-validate inside [[Server Actions]] / Route Handlers.

💡 Common uses: auth gating, geo/locale rewrites, feature flags, request ID injection, CORS preflight.

## Tags
[[Nextjs]] [[App Router]] [[Edge]] [[Auth]]
