# 04 — Server Actions

🔑 Server Actions are async functions marked with `"use server"` that run on the server but can be called like normal functions from Client Components — typically from `<form action={...}>`.

Source: https://nextjs.org/docs/app/api-reference/directives/use-server

## Two Ways to Declare
**File-level** — every export is a Server Action:
```ts
// app/actions.ts
'use server'

import { db } from '@/lib/db'
import { auth } from '@/lib/auth'
import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')
  const title = formData.get('title') as string
  await db.post.create({ data: { title, userId: session.user.id } })
  revalidatePath('/posts')
}
```

**Inline** — declare inside a Server Component:
```tsx
export default function Page() {
  async function save(fd: FormData) {
    'use server'
    await db.post.create({ data: { title: fd.get('title') as string } })
    revalidatePath('/posts')
  }
  return <form action={save}><input name="title" /></form>
}
```

## Calling from a Client Component
```tsx
'use client'
import { createPost } from '../actions'
// either as form action or directly:
<form action={createPost}>...</form>
```

## Post-Mutation APIs
| Function | Purpose |
|---|---|
| `revalidatePath('/posts')` | Invalidate route cache for that path |
| `revalidateTag('posts')` | Invalidate all caches tagged `posts` |
| `redirect('/posts/123')` | Throw redirect — must call after the mutation |
| `cookies()` / `headers()` | Read or set request-scoped values |

⚠️ **Always re-authenticate inside the action.** A Server Action is just a POST endpoint — anyone can call it. Don't trust the route or [[Middleware]] alone.

💡 Return only the data the UI needs. The return value is serialized over the wire — never return raw DB rows.

🧪 Test by calling the exported action with a real `FormData` instance — it's just an async function.

## Tags
[[Nextjs]] [[Server Actions]] [[RSC]] [[App Router]]
