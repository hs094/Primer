# 11 — Metadata and SEO

🔑 Export a `metadata` object (static) or `generateMetadata` function (dynamic) from `layout.tsx` / `page.tsx` — Next.js injects the `<head>` tags. File-based conventions cover icons, OG images, sitemap, and robots.

Source: https://nextjs.org/docs/app/getting-started/metadata-and-og-images

## Static Metadata
```tsx
// app/blog/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: { default: 'Blog', template: '%s | Acme' },
  description: 'Posts and notes',
  openGraph: { title: 'Blog', images: ['/og.png'] },
}
```

## Dynamic — `generateMetadata`
```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata(
  { params }: { params: Promise<{ slug: string }> }
): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)
  return { title: post.title, description: post.excerpt }
}
```

⚠️ `generateMetadata` and `Page` often need the same data — wrap the loader in `React.cache()` so the request runs once.

## File Conventions
| File | Output |
|---|---|
| `app/favicon.ico`, `icon.png`, `apple-icon.png` | Favicons / Apple icons |
| `app/opengraph-image.{png,jpg}` or `.tsx` | OG image (static or generated) |
| `app/twitter-image.{png,jpg}` or `.tsx` | Twitter Card image |
| `app/sitemap.ts` | `MetadataRoute.Sitemap` |
| `app/robots.ts` | `MetadataRoute.Robots` |

## Generated OG Image
```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og'
export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'

export default async function Image({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug)
  return new ImageResponse(<div style={{ fontSize: 96 }}>{post.title}</div>)
}
```

## `sitemap.ts` / `robots.ts`
```ts
import type { MetadataRoute } from 'next'
export default function sitemap(): MetadataRoute.Sitemap {
  return [{ url: 'https://acme.com', lastModified: new Date(), priority: 1 }]
}
```

💡 JSON-LD: render a `<script type="application/ld+json">` directly in the page — Next doesn't have a dedicated API for it, plain JSX is the recommendation.

## Tags
[[Nextjs]] [[SEO]] [[App Router]] [[OpenGraph]]
