# 08 — Parallel and Intercepting Routes

🔑 `@slot` folders render multiple pages **simultaneously** in one layout; `(.)`-prefixed folders **intercept** another route to render it inline (e.g. as a modal).

Sources:
- https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes
- https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes

## Parallel — `@slot`
```
app/
  layout.tsx
  @analytics/page.tsx
  @team/page.tsx
  page.tsx
```
```tsx
export default function Layout({
  children, analytics, team,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  team: React.ReactNode
}) {
  return <>{children}{team}{analytics}</>
}
```
Slots don't change the URL. Each renders, errors, and streams independently.

⚠️ Every slot needs `default.tsx` — fallback on hard navigation when the slot can't match the URL. Without it, 404.

## Intercepting — `(.)` `(..)` `(...)`
| Pattern | Intercepts |
|---|---|
| `(.)foo` | sibling segment `foo` |
| `(..)foo` | one level up |
| `(..)(..)foo` | two levels up |
| `(...)foo` | from root `app/` |

## Modal Pattern (parallel + intercepting)
```
app/
  layout.tsx               # renders {children}{auth}
  @auth/
    default.tsx            # returns null
    (.)login/page.tsx      # <Modal><Login /></Modal>
  login/page.tsx           # full-page version on refresh
```
Soft nav from `<Link href="/login">` opens the modal; refresh / shareable URL renders the full page. Close with `router.back()`.

🧪 Reference: `vercel-labs/nextgram` photo-modal example.

## Tags
[[Nextjs]] [[App Router]] [[Routing]] [[Streaming]]
