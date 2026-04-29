# 10 — Image and Font

🔑 `next/image` and `next/font` ship Core Web Vitals fixes out of the box — auto-sizing, lazy loading, format negotiation, and zero CLS for fonts via self-hosting.

Sources:
- https://nextjs.org/docs/app/getting-started/images
- https://nextjs.org/docs/app/api-reference/components/font

## `next/image`
```tsx
import Image from 'next/image'
import profile from './profile.png'

// Static import → width/height/blurDataURL inferred
<Image src={profile} alt="Profile" placeholder="blur" />

// Local path from /public
<Image src="/hero.png" alt="" width={1200} height={630} priority />

// Remote — must whitelist the host
<Image src="https://cdn.acme.com/x.png" alt="" width={400} height={400} />
```

What you get for free:
- Modern formats (AVIF/WebP) negotiated per-browser.
- Responsive `srcset` based on `sizes`.
- Native `loading="lazy"` unless `priority` is set (use for LCP image).
- Reserved space → no Cumulative Layout Shift.

```ts
// next.config.ts — required for remote hosts
const config = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.acme.com', pathname: '/**' }],
  },
}
```

⚠️ Always set `alt`, and set `priority` on the LCP image only (typically one per page).

## `next/font`
```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], display: 'swap', variable: '--font-sans' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html lang="en" className={inter.variable}><body>{children}</body></html>
}
```
- Fonts are downloaded at build, **self-hosted** with your assets — no Google request from the browser, no FOUT.
- Variable fonts skip the `weight` array.
- Local fonts: `import localFont from 'next/font/local'` + `src: './my.woff2'`.

💡 Use `variable: '--font-sans'` and reference via CSS variables to integrate with Tailwind's `@theme` block or `tailwind.config.js`.

🧪 `display: 'swap'` is the default; set `'optional'` for the strictest CLS guarantee on slow networks.

## Tags
[[Nextjs]] [[Performance]] [[Images]] [[Fonts]]
