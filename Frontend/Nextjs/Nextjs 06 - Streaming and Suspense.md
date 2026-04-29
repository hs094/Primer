# 06 — Streaming and Suspense

🔑 Streaming sends HTML to the browser in chunks as each Suspense boundary resolves — the first paint shows the static shell instantly, slow data fills in.

Source: https://nextjs.org/docs/app/getting-started/fetching-data#streaming

## Two Entry Points
| API | Granularity |
|---|---|
| `loading.tsx` | Whole route segment — auto-wraps `page.tsx` in `<Suspense>` |
| `<Suspense>` | Per-component, anywhere in the tree |

## `loading.tsx`
```tsx
// app/blog/loading.tsx
export default function Loading() {
  return <BlogSkeleton />
}
```
Renders immediately on navigation; swapped for `page.tsx` once the page resolves. Lives at any segment level.

## `<Suspense>` for Granular Holes
```tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <main>
      <Header />                                    {/* instant */}
      <Suspense fallback={<PostsSkeleton />}>
        <Posts />                                   {/* awaits DB */}
      </Suspense>
    </main>
  )
}

async function Posts() {
  const posts = await db.post.findMany()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

## When to Suspend
- Slow / uncertain data (DB, third-party API).
- Anything below "above-the-fold" — let the shell paint first.
- Per-card loading states in a grid (each card its own boundary).

## Streaming Data to Client Components
Don't `await` in the parent — pass the promise; resolve with `use()` on the client:
```tsx
// Server
export default function Page() {
  const posts = getPosts()  // not awaited
  return <Suspense fallback={<Skel />}><Posts posts={posts} /></Suspense>
}
// Client
'use client'
import { use } from 'react'
export default function Posts({ posts }: { posts: Promise<Post[]> }) {
  return <ul>{use(posts).map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

⚠️ `loading.tsx` does **not** cover a layout that itself accesses `cookies()` / `headers()` — that blocks navigation. Wrap the dynamic access in its own `<Suspense>`.

💡 [[Cache Components]] composes naturally: cache the shell, suspend the dynamic chunks.

## Tags
[[Nextjs]] [[Streaming]] [[RSC]] [[Suspense]]
