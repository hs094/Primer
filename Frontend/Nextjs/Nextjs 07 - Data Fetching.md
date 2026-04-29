# 07 — Data Fetching

🔑 Server Components fetch directly with `async`/`await` — no `getServerSideProps`. Use `fetch` with Next's caching extensions, or hit an ORM/DB.

Source: https://nextjs.org/docs/app/getting-started/fetching-data

## Server Component Fetch
```tsx
export default async function Page() {
  const res = await fetch('https://api.example.com/posts')
  const posts: Post[] = await res.json()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

## Caching Extensions
```ts
fetch(url, { next: { revalidate: 3600 } })  // cache for 1h
fetch(url, { next: { tags: ['posts'] } })   // tag-based invalidation
fetch(url, { cache: 'no-store' })           // always fresh
fetch(url, { cache: 'force-cache' })        // never refetch

import { revalidateTag } from 'next/cache'
revalidateTag('posts')
```
Identical `GET` fetches are auto-memoized within one render pass.

## Parallel vs Sequential
```tsx
// Sequential (waterfall)
const artist = await getArtist(id)
const albums = await getAlbums(id)

// Parallel
const [artist, albums] = await Promise.all([getArtist(id), getAlbums(id)])
```

## Async `params` / `searchParams`
```tsx
export default async function Page({
  params,
  searchParams,
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
export async function generateStaticParams() {
  const posts = await db.post.findMany({ select: { slug: true } })
  return posts.map(p => ({ slug: p.slug }))
}
export const dynamicParams = false  // unknown slugs → 404
```

⚠️ With [[Cache Components]], `generateStaticParams` must return ≥1 entry — empty arrays cause a build error.

💡 Wrap shared loaders in `React.cache()` to dedupe across `generateMetadata` and `Page`.

## Tags
[[Nextjs]] [[App Router]] [[RSC]] [[Cache Components]]
