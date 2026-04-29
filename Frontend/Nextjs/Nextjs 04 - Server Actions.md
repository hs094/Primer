# 04 — Server Actions

🔑 Async functions marked `"use server"` run on the server but are callable from Client Components — typically via `<form action={...}>`.

Source: https://nextjs.org/docs/app/api-reference/directives/use-server

## File-level Declaration
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

## Inline (Server Component)
```tsx
async function save(fd: FormData) {
  'use server'
  await db.post.create({ data: { title: fd.get('title') as string } })
  revalidatePath('/posts')
}
return <form action={save}><input name="title" /></form>
```

## Calling from Client
```tsx
'use client'
import { createPost } from '../actions'
<form action={createPost}>...</form>
```

## Post-Mutation APIs
| Function | Purpose |
|---|---|
| `revalidatePath('/posts')` | Invalidate route cache |
| `revalidateTag('posts')` | Invalidate tagged caches |
| `redirect('/posts/123')` | Throw redirect after mutation |
| `cookies()` / `headers()` | Read/set request-scoped values |

⚠️ A Server Action is just a POST endpoint — re-authenticate inside every action. Never trust the route or [[Middleware]] alone.

💡 Return only what the UI needs; values are serialized over the wire.

## Tags
[[Nextjs]] [[Server Actions]] [[RSC]] [[App Router]]
