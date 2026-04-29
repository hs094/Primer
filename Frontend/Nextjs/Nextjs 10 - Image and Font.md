# 10 — Image and Font

🔑 `next/image` and `next/font` ship Core Web Vitals fixes out of the box — auto-sizing, lazy loading, format negotiation, and zero CLS via self-hosting.

Sources:
- https://nextjs.org/docs/app/getting-started/images
- https://nextjs.org/docs/app/api-reference/components/font

## `next/image`
```tsx
import Image from 'next/image'
import profile from './profile.png'

// Static import → width/height/blurDataURL inferred
<Image src={profile} alt="Profile" placeholder="blur" />

// /public path
<Image src="/hero.png" alt="" width={1200} height={630} priority />

// Remote — host must be whitelisted
<Image src="https://cdn.acme.com/x.png" alt="" width={400} height={400} />
```
Free: AVIF/WebP, responsive `srcset`, lazy loading, reserved space (no CLS).

```ts
// next.config.ts
const config = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'cdn.acme.com', pathname: '/**' }],
  },
}
```

⚠️ Always set `alt`. Set `priority` only on the LCP image (typically one per page).

## `next/font`
```tsx
// app/layout.tsx
import { Inter } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], display: 'swap', variable: '--font-sans' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html lang="en" className={inter.variable}><body>{children}</body></html>
}
```
- Downloaded at build, **self-hosted** with assets — no Google request from the browser, no FOUT.
- Variable fonts skip `weight`.
- Local: `import localFont from 'next/font/local'` + `src: './my.woff2'`.

💡 `variable: '--font-sans'` integrates cleanly with Tailwind's `@theme` block.

🧪 `display: 'swap'` is default; use `'optional'` for the strictest CLS guarantee.

## Tags
[[Nextjs]] [[Performance]] [[Images]] [[Fonts]]
