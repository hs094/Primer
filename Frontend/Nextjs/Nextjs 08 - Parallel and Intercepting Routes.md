# 08 — Parallel and Intercepting Routes

🔑 Two file-system tricks for advanced UI: `@slot` folders render multiple pages **simultaneously** in one layout; `(.)`-prefixed folders **intercept** another route to render it inline (e.g. as a modal).

Sources:
- https://nextjs.org/docs/app/api-reference/file-conventions/parallel-routes
- https://nextjs.org/docs/app/api-reference/file-conventions/intercepting-routes

## Parallel Routes — `@slot`
A folder prefixed with `@` becomes a named slot, passed as a prop to the parent layout:
```
app/
  layout.tsx
  @analytics/page.tsx
  @team/page.tsx
  page.tsx
```
```tsx
// app/layout.tsx
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

⚠️ Every slot needs a `default.tsx` — used as fallback on hard navigation when the slot can't match the URL. Without it, you get a 404.

💡 Conditional slots: render `admin` vs `user` from the same layout based on session.

## Intercepting Routes — `(.)` `(..)` `(...)`
Mirror the relative-path syntax, but for route segments:
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
  login/page.tsx           # full-page version (refresh / direct link)
```
Soft nav to `/login` from a `<Link>` opens the modal; refresh or shareable URL renders the full page. Closing calls `router.back()`.

🧪 See `vercel-labs/nextgram` for a complete photo-modal example.

## Tags
[[Nextjs]] [[App Router]] [[Routing]] [[Streaming]]
