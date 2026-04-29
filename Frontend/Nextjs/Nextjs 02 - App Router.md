# 02 — App Router

🔑 Folders inside `app/` are URL segments; special files (`page`, `layout`, `loading`, `error`, `not-found`) are the route's UI primitives.

Source: https://nextjs.org/docs/app/getting-started/project-structure

## Special Files
| File | Role |
|---|---|
| `page.tsx` | Public route — without it, the segment is private |
| `layout.tsx` | Wraps children; persists across nav, doesn't re-mount |
| `loading.tsx` | React Suspense fallback, auto-wraps `page.tsx` |
| `error.tsx` | Error boundary; must be a Client Component |
| `not-found.tsx` | UI for `notFound()` calls and unmatched routes |
| `route.ts` | API endpoint (GET/POST/...) instead of a page |
| `template.tsx` | Like layout, but re-mounts each navigation |

## Component Hierarchy
`layout` → `template` → `error` → `loading` → `not-found` → `page`. Nested segments nest these recursively.

## Routing Patterns
| Path | URL |
|---|---|
| `app/blog/page.tsx` | `/blog` |
| `app/blog/[slug]/page.tsx` | `/blog/hello` |
| `app/shop/[...slug]/page.tsx` | `/shop/a/b/c` (catch-all) |
| `app/docs/[[...slug]]/page.tsx` | `/docs` *and* `/docs/a/b` (optional catch-all) |
| `app/(marketing)/about/page.tsx` | `/about` — `(marketing)` is a route group, omitted from URL |
| `app/_components/Btn.tsx` | not routable — `_` prefix opts out |

## Page Signature
```tsx
// app/blog/[slug]/page.tsx
export default async function Page({
  params,
}: { params: Promise<{ slug: string }> }) {
  const { slug } = await params  // params is async in v15+
  return <article>{slug}</article>
}
```

⚠️ `params` and `searchParams` are **Promises** — must be awaited before access.

💡 Use route groups `(name)` to share a layout across a subset of routes without changing URLs. Multiple root layouts: drop the top-level `layout.tsx` and add one per group.

## Tags
[[Nextjs]] [[App Router]] [[RSC]] [[Routing]]
