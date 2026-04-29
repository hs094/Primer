# 12 — Deployment

🔑 Next.js runs anywhere a Node.js process or Docker container runs. Vercel is zero-config; everything else uses adapters or `output: 'standalone'`.

Source: https://nextjs.org/docs/app/getting-started/deploying

## Targets
| Target | Feature support | When |
|---|---|---|
| **Vercel** | All (verified adapter) | Default, auto ISR/edge |
| **Node.js** | All | `next build && next start` on any host |
| **Docker** | All | `output: 'standalone'` for tiny images |
| **Static export** | Limited | `output: 'export'` — no SSR, no Server Actions |
| **Adapters** | Varies | Bun verified; Cloudflare/Netlify ship integrations |

## Standalone Output (Docker)
```ts
const config = { output: 'standalone' }
```
`next build` writes `.next/standalone/` with a self-contained Node server + traced deps (~150 MB image vs ~1 GB).

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY .next/standalone ./
COPY .next/static ./.next/static
COPY public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

## Edge Runtime (per-route)
```ts
export const runtime = 'edge'
```
Restricted Node API surface, lower cold start, V8 isolates near the user. Use for tiny handlers, [[Middleware]], geo-aware responses.

## ISR
```ts
import { revalidatePath, revalidateTag } from 'next/cache'
revalidateTag('posts')
```
Time-based: `fetch(url, { next: { revalidate: 3600 } })` or `cacheLife('hours')` inside a [[Cache Components]] scope.

## Other Platforms
**OpenNext** (`open-next.js.org`) for Cloudflare Workers / AWS Lambda. Templates also for Amplify, Firebase, Render, Railway, Fly.io.

⚠️ Static export disables Server Actions, ISR, image optimization, middleware/proxy, and dynamic routes without `generateStaticParams`.

💡 Cache Components entries are in-memory; on serverless they don't persist across instances. Use `cacheHandlers` to plug in Redis/KV when self-hosting at scale.

## Tags
[[Nextjs]] [[Deployment]] [[Vercel]] [[Docker]] [[Edge]]
