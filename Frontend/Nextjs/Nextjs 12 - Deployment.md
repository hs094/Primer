# 12 — Deployment

🔑 Next.js runs anywhere a Node.js process or Docker container runs. Vercel is the zero-config target; everything else uses adapters or `output: 'standalone'`.

Source: https://nextjs.org/docs/app/getting-started/deploying

## Targets at a Glance
| Target | Feature support | When |
|---|---|---|
| **Vercel** | All (verified adapter) | Default, zero-config, automatic ISR/edge |
| **Node.js server** | All | `next build && next start` on any host |
| **Docker** | All | `output: 'standalone'` for tiny images |
| **Static export** | Limited | `output: 'export'` — no SSR, no Server Actions |
| **Adapters** | Varies | Bun (verified), Cloudflare/Netlify (own integrations) |

## Standalone Output (Docker)
```ts
// next.config.ts
const config = { output: 'standalone' }
```
`next build` writes `.next/standalone/` with a self-contained Node server + traced deps — typical image goes from ~1 GB to ~150 MB.

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
// app/api/hello/route.ts
export const runtime = 'edge'
```
Restricted Node API surface, but lower cold start and runs in V8 isolates close to the user. Use for tiny handlers, [[Middleware]] / proxy logic, geo-aware responses.

## ISR — Incremental Static Regeneration
On-demand revalidation works on any Node deploy:
```ts
// In a Server Action or Route Handler
import { revalidatePath, revalidateTag } from 'next/cache'
revalidateTag('posts')
```
Time-based: `fetch(url, { next: { revalidate: 3600 } })` or `cacheLife('hours')` inside a [[Cache Components]] scope.

## Other Platforms
- **OpenNext** (`open-next.js.org`) — community adapter for Cloudflare Workers and AWS Lambda.
- **AWS Amplify**, **Firebase App Hosting**, **Render**, **Railway**, **Fly.io** — all have starter templates under `github.com/nextjs`.

⚠️ Static export disables Server Actions, ISR, image optimization (without a custom loader), middleware/proxy, and dynamic routes without `generateStaticParams`. It's HTML/CSS/JS only.

💡 [[Cache Components]] entries are stored in-memory; on serverless, entries don't persist across instances. Use `cacheHandlers` to plug in Redis/KV when you self-host at scale.

## Tags
[[Nextjs]] [[Deployment]] [[Vercel]] [[Docker]] [[Edge]]
