# 06 — Streaming and Suspense

🔑 Streaming sends HTML in chunks as each Suspense boundary resolves — the static shell paints instantly while slow data fills in.

Source: https://nextjs.org/docs/app/getting-started/fetching-data#streaming

## Two Entry Points
| API | Granularity |
|---|---|
| `loading.tsx` | Whole route segment — auto-wraps `page.tsx` in `<Suspense>` |
| `<Suspense>` | Per-component, anywhere in the tree |

## `loading.tsx`
```tsx
// app/blog/loading.tsx
export default function Loading() { return <BlogSkeleton /> }
```
Renders immediately on navigation; swapped for `page.tsx` once it resolves.

## Granular `<Suspense>`
```tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <main>
      <Header />
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />
      </Suspense>
    </main>
  )
}

async function Posts() {
  const posts = await db.post.findMany()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

## Streaming to Client Components
Pass an unawaited promise; resolve with `use()`:
```tsx
// Server
const posts = getPosts()  // not awaited
return <Suspense fallback={<Skel />}><Posts posts={posts} /></Suspense>

// Client
'use client'
import { use } from 'react'
export default function Posts({ posts }: { posts: Promise<Post[]> }) {
  return <ul>{use(posts).map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

⚠️ `loading.tsx` doesn't cover a layout that itself reads `cookies()`/`headers()` — wrap that access in its own `<Suspense>`.

💡 Composes with [[Cache Components]]: cache the shell, suspend dynamic chunks.

## Tags
[[Nextjs]] [[Streaming]] [[RSC]] [[Suspense]]
