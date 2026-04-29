# 07 — Data Fetching

🔑 Server Components fetch directly with `async`/`await` — no `getServerSideProps`, no loaders. Use `fetch` with Next's caching extensions, or hit an ORM/DB.

Source: https://nextjs.org/docs/app/getting-started/fetching-data

## Server Component Fetch
```tsx
export default async function Page() {
  const data = await fetch('https://api.example.com/posts')
  const posts = await data.json()
  return <ul>{posts.map((p: Post) => <li key={p.id}>{p.title}</li>)}</ul>
}
```

## Next's `fetch` Extensions
```ts
// Cache for 1 hour
fetch(url, { next: { revalidate: 3600 } })

// Tag-based invalidation
fetch(url, { next: { tags: ['posts'] } })
// Then anywhere on the server:
import { revalidateTag } from 'next/cache'
revalidateTag('posts')

// Force fresh on every request
fetch(url, { cache: 'no-store' })
// Force cache forever
fetch(url, { cache: 'force-cache' })
```

Identical `GET` fetches are auto-memoized within a single render pass — no need to prop-drill.

## Parallel vs Sequential
```tsx
// Sequential — slow (waterfall)
const artist = await getArtist(id)
const albums = await getAlbums(id)

// Parallel — kick off both, await together
const [artist, albums] = await Promise.all([getArtist(id), getAlbums(id)])
```

## Async `params` / `searchParams`
```tsx
export default async function Page({
  params, searchParams,
}: {
  params: Promise<{ slug: string }>
  searchParams: Promise<{ q?: string }>
}) {
  const { slug } = await params
  const { q } = await searchParams
}
```

## Static Generation
```ts
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true } })
  return posts.map(p => ({ slug: p.slug }))
}
// Block unknown slugs at runtime → 404
export const dynamicParams = false
```

⚠️ With [[Cache Components]] enabled, `generateStaticParams` must return at least one entry — empty arrays cause a build error.

💡 Wrap shared loaders in `React.cache()` to dedupe across `generateMetadata` and `Page` in the same request.

## Tags
[[Nextjs]] [[App Router]] [[RSC]] [[Cache Components]]
