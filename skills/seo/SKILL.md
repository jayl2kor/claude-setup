---
name: seo
description: This skill should be used when the user asks to optimize SEO, add meta tags, implement structured data, fix Core Web Vitals, set up sitemaps or robots.txt, add canonical URLs, configure hreflang for internationalization, improve search rankings, add OpenGraph or Twitter Card tags, or any task involving 검색 최적화, SEO 적용, or search engine discoverability.
version: 1.0.0
---

# SEO

Expert-level SEO implementation covering technical fundamentals, Core Web Vitals, structured data, and framework-specific patterns. Applies to Next.js (App Router and Pages Router), React SPAs, and any HTML-based site.

## When to Activate

- Adding or auditing `<title>`, `<meta name="description">`, canonical, or robots meta tags
- Implementing structured data (JSON-LD schema.org types)
- Fixing Core Web Vitals regressions (LCP, INP, CLS)
- Setting up `robots.txt`, XML sitemaps, or sitemap index files
- Configuring canonical URLs or hreflang for multi-language sites
- Adding OpenGraph / Twitter Card tags for social sharing
- Evaluating SSR vs SSG vs CSR for SEO impact
- Interpreting Google Search Console reports or Lighthouse SEO audits

---

## Technical SEO Fundamentals

Three phases determine whether a page ranks. Each has different failure modes:

| Phase | What Google Does | Common Failure |
|---|---|---|
| **Crawling** | Googlebot fetches page HTML via HTTP | `robots.txt` blocking crawler; server 5xx errors; crawl budget exhausted on low-value URLs |
| **Indexing** | Parses content, extracts signals, picks canonical | `noindex` directive; duplicate content without canonical; JavaScript-only rendering that Googlebot can't execute |
| **Ranking** | Scores against hundreds of signals | Thin/duplicate content; poor Core Web Vitals; missing E-E-A-T signals; no structured data |

**Core ranking signals to address first:**

1. **Content relevance** — title, headings, body copy all match search intent
2. **Page experience** — Core Web Vitals at 75th percentile (see below)
3. **Structured data** — helps Google understand entity type and eligibility for rich results
4. **Authority signals** — inbound links, author credibility (`author` schema with URL)
5. **Crawl accessibility** — `robots.txt` permits indexing; sitemap submitted to Search Console

---

## Core Web Vitals Quick Reference

All thresholds measured at the **75th percentile** across mobile and desktop.

| Metric | Measures | Good | Needs Work | Poor | Top Fix |
|---|---|---|---|---|---|
| **LCP** | Loading — largest visible element paint time | ≤ 2.5s | 2.5–4.0s | > 4.0s | Add `fetchpriority="high"` to hero image; `<link rel="preload">` for above-fold images |
| **INP** | Responsiveness — worst interaction delay | ≤ 200ms | 200–500ms | > 500ms | Break long event handlers into tasks with `setTimeout`; reduce DOM size |
| **CLS** | Visual stability — layout shift score | ≤ 0.1 | 0.1–0.25 | > 0.25 | Set `width`/`height` on all images; `font-display: optional` or `size-adjust` on web fonts |

INP replaced FID as a Core Web Vital in March 2024. FID data in older Search Console reports is historical only.

---

## Next.js Metadata API — Most Common Pattern

### App Router (recommended)

Static metadata in `layout.tsx` sets site-wide defaults with a title template:

```tsx
// app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),
  title: {
    default: 'Acme',
    template: '%s | Acme',   // child pages set title: 'About' → "About | Acme"
  },
  description: 'Default site description (150–160 chars)',
  openGraph: {
    siteName: 'Acme',
    locale: 'en_US',
    type: 'website',
  },
  robots: { index: true, follow: true },
}
```

Dynamic metadata for data-driven pages (e.g. `/products/[id]`):

```tsx
// app/products/[id]/page.tsx
import type { Metadata } from 'next'

export async function generateMetadata(
  { params }: { params: Promise<{ id: string }> }
): Promise<Metadata> {
  const { id } = await params
  const product = await fetch(`https://api.example.com/products/${id}`).then(r => r.json())

  return {
    title: product.name,                          // → "Acme Widget | Acme"
    description: product.description.slice(0, 155),
    alternates: { canonical: `/products/${id}` },
    openGraph: {
      title: product.name,
      images: [{ url: product.imageUrl, width: 1200, height: 630 }],
      type: 'website',
    },
  }
}
```

Key constraints: `metadata` exports are **Server Components only**. `generateMetadata` fetch calls are automatically memoized — safe to call the same endpoint in the page component without double-fetching.

### Pages Router

```tsx
// pages/products/[id].tsx
import Head from 'next/head'

export default function ProductPage({ product }) {
  return (
    <>
      <Head>
        <title>{product.name} | Acme</title>
        <meta name="description" content={product.description.slice(0, 155)} />
        <link rel="canonical" href={`https://example.com/products/${product.id}`} />
        <meta property="og:title" content={product.name} />
        <meta property="og:image" content={product.imageUrl} />
        <meta name="twitter:card" content="summary_large_image" />
      </Head>
      {/* page content */}
    </>
  )
}
```

Use the `key` prop to prevent duplicate `<meta>` tags when multiple `<Head>` blocks are present.

---

## Structured Data Quick Start (JSON-LD)

Inject in `<script type="application/ld+json">` inside `<head>`. Google prefers JSON-LD over Microdata or RDFa.

### Article / Blog Post

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "How to Optimize Core Web Vitals",
  "datePublished": "2024-06-01T09:00:00Z",
  "dateModified": "2024-09-15T12:00:00Z",
  "author": [{ "@type": "Person", "name": "Jane Doe", "url": "https://example.com/authors/jane" }],
  "image": ["https://example.com/images/article-16x9.jpg"],
  "publisher": { "@type": "Organization", "name": "Acme", "logo": { "@type": "ImageObject", "url": "https://example.com/logo.png" } }
}
```

### Product

```json
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Acme Widget",
  "image": "https://example.com/widget.jpg",
  "description": "A quality widget for all occasions.",
  "sku": "WIDGET-42",
  "brand": { "@type": "Brand", "name": "Acme" },
  "offers": {
    "@type": "Offer",
    "price": "29.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock",
    "url": "https://example.com/products/widget"
  },
  "aggregateRating": { "@type": "AggregateRating", "ratingValue": "4.7", "reviewCount": "128" }
}
```

### BreadcrumbList (for sub-pages)

```json
{
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Home", "item": "https://example.com" },
    { "@type": "ListItem", "position": 2, "name": "Products", "item": "https://example.com/products" },
    { "@type": "ListItem", "position": 3, "name": "Acme Widget" }
  ]
}
```

Validate with [Rich Results Test](https://search.google.com/test/rich-results) before deploying.

---

## Common SEO Mistakes Checklist

**Crawling / Indexing**
- [ ] `robots.txt` blocks pages that should be indexed (check `/robots.txt` in browser)
- [ ] `noindex` left on staging environment and shipped to production
- [ ] JavaScript-rendered content not visible in "View Page Source" (needs SSR/SSG)
- [ ] Canonical tag missing on paginated, filtered, or sorted URL variants
- [ ] Canonical points to the wrong URL (e.g. HTTP when site is HTTPS)
- [ ] Sitemap not submitted to Google Search Console

**Meta Tags**
- [ ] `<title>` missing or duplicated across pages
- [ ] Meta description over 160 characters (gets truncated in SERPs)
- [ ] `og:image` is a relative URL — must be absolute with protocol
- [ ] `og:image` not 1200×630px — smaller images display as thumbnails
- [ ] `twitter:card` missing — defaults to `summary` with tiny image

**Structured Data**
- [ ] JSON-LD embedded in a Client Component that never reaches the server-rendered HTML
- [ ] `datePublished`/`dateModified` in wrong format (must be ISO 8601)
- [ ] `author.url` missing — reduces author entity credibility signals
- [ ] Product `offers` missing `priceCurrency` or `availability`

**Performance (CWV)**
- [ ] Hero image loaded without `fetchpriority="high"` or `priority` prop (Next.js `<Image>`)
- [ ] Images missing `width`/`height` attributes — causes CLS
- [ ] Web fonts loaded without `font-display: swap` or `optional` — causes invisible text flash
- [ ] Long event handlers blocking main thread > 200ms — fails INP

**Semantic HTML**
- [ ] Multiple `<h1>` elements on one page
- [ ] `<img>` without `alt` text (decorative images need `alt=""`)
- [ ] Page lacks landmark elements (`<main>`, `<nav>`, `<header>`, `<footer>`)
- [ ] Heading levels skipped (e.g. `<h1>` → `<h3>`, no `<h2>`)

---

## References

- [references/technical-seo.md](references/technical-seo.md) — robots.txt patterns, XML sitemap config, canonical strategy, hreflang, full Next.js Metadata API examples, SSR/SSG/ISR decision criteria, OpenGraph + Twitter Card full tag list, semantic HTML rules
- [references/performance-seo.md](references/performance-seo.md) — LCP/INP/CLS deep dive with specific fixes, Lighthouse SEO audit interpretation, Google Search Console key metrics
