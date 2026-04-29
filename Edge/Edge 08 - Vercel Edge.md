# 08 — Vercel Edge

🔑 Vercel Edge is V8 isolates (same family as Workers) wired into Next.js. Two flavours: **Edge Functions** (route handlers) and **Edge Middleware** (runs before every matched request).

Source: https://vercel.com/docs/functions/runtimes/edge

## Edge Function vs Edge Middleware

| | Edge Function | Edge Middleware |
|---|---|---|
| When | On request to a route | Before every matched route |
| Returns | A `Response` | `NextResponse.next()` / rewrite / redirect |
| Use | API endpoint, streaming SSR | Auth, geo redirects, A/B, header rewrites |
| File | `app/api/*/route.ts` | `middleware.ts` at project root |

## Declaring the runtime

```ts
// app/api/hello/route.ts
export const runtime = "edge";          // 'nodejs' is default
export const preferredRegion = ["iad1", "fra1"];

export function GET(req: Request) {
  return new Response(`hello from ${process.env.VERCEL_REGION}`);
}
```

## Edge Middleware

```ts
// middleware.ts
import { NextResponse, type NextRequest } from "next/server";

export const config = { matcher: ["/dashboard/:path*"] };

export function middleware(req: NextRequest) {
  if (!req.cookies.get("session")) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
  return NextResponse.next();
}
```

## Runtime constraints

- 🔑 **Web standard APIs only** — `fetch`, `Request`, `Response`, `crypto`, `TextEncoder`, streams.
- ⚠️ **No FS, no `child_process`, no raw TCP/UDP, no native modules.**
- ⚠️ `eval` and `new Function(string)` are blocked. WebAssembly only via static `import`.
- A subset of `node:` modules works: `async_hooks` (`AsyncLocalStorage`), `events`, `buffer`, `assert`, `util`.
- 💡 Limits: code size 1–4 MB gzipped; **must start streaming within 25 s**, can stream up to 300 s.

## 💡 Vercel guidance, 2026

- Vercel now **recommends migrating Edge Functions back to the Node.js runtime on Fluid Compute** for most workloads — same global routing, fewer constraints. Edge runtime stays best for middleware and ultra-light response shaping. See [[Edge 09 - Edge Limits and Pitfalls]].

## Tags

[[Edge]] [[Vercel]] [[Next.js]] [[V8]] [[middleware]]
