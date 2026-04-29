# 03 — React Server Components

🔑 In the App Router, every component is a **Server Component by default** — no JS ships to the client unless you opt in with `"use client"`.

Source: https://nextjs.org/docs/app/getting-started/server-and-client-components

## Server vs Client
| Server Component | Client Component |
|---|---|
| `async`/`await` data fetching | `useState`, `useEffect`, event handlers |
| Direct DB / ORM / fs access | Browser APIs (`window`, `localStorage`) |
| Secrets stay server-side | React context providers |
| Zero client JS | Hydrated, ships JS bundle |

## The `"use client"` Boundary
```tsx
// app/ui/counter.tsx
'use client'
import { useState } from 'react'

export default function Counter() {
  const [n, setN] = useState(0)
  return <button onClick={() => setN(n + 1)}>{n}</button>
}
```
The directive marks the **boundary**: the file *and everything it imports* becomes part of the client bundle. You don't add it to every leaf.

## Composition Pattern
Server Components can render Client Components, and pass Server Components as `children` *into* Client Components — but a Client Component cannot directly import a Server Component.

```tsx
// Server Component
export default async function Page() {
  const post = await db.post.find()
  return <LikeButton likes={post.likes} />  // hands serialized props off
}
```

⚠️ Props passed Server → Client must be **serializable**: primitives, plain objects, arrays, Dates, Maps, Sets, JSX. No class instances, functions (except Server Actions), or `URL` instances.

💡 Push `"use client"` as deep as possible. A static `<Layout>` with one interactive `<Search>` should keep the layout server-side and only mark `Search` as client.

🧪 Install `server-only` to throw a build error if a server module is imported from the client bundle (protects API keys / secrets).

## Tags
[[Nextjs]] [[RSC]] [[App Router]] [[React]]
