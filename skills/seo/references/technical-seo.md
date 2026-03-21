# Technical SEO Reference

Deep-dive reference for robots.txt, sitemaps, canonicals, hreflang, Next.js metadata, rendering strategies, social tags, and semantic HTML. See [../SKILL.md](../SKILL.md) for the quick-reference summary.

---

## robots.txt

Place at the root of the domain: `https://example.com/robots.txt`. Googlebot fetches this before crawling any other URL.

### Syntax

```
User-agent: *
Disallow: /admin/
Disallow: /api/
Allow: /api/public/

User-agent: Googlebot
Disallow: /no-google/

Sitemap: https://example.com/sitemap.xml
Sitemap: https://example.com/sitemap-blog.xml
```

### Key Rules

- `User-agent: *` applies to all crawlers; more specific rules override it for named bots.
- `Disallow:` with an empty value means allow all — `Disallow:` (blank) is a no-op.
- `Allow:` overrides a broader `Disallow` for a specific path prefix.
- `Crawl-delay:` is respected by Bing and some others but **ignored by Googlebot**. Use Google Search Console's crawl rate settings instead.
- Multiple `Sitemap:` lines are valid.

### What robots.txt Cannot Do

- It **cannot prevent indexing** of a page someone links to. Use `noindex` for that.
- It **cannot protect sensitive data** — blocked URLs still appear in search results if linked externally (without a snippet). Use authentication or server-level access control.
- Different crawlers may interpret syntax differently. Test with Google's [robots.txt Tester](https://search.google.com/search-console/robots-testing-tool).

### Common Patterns

```
# Block all crawlers from entire site (maintenance mode)
User-agent: *
Disallow: /

# Block duplicate faceted navigation URLs
User-agent: *
Disallow: /products?sort=
Disallow: /products?filter=
Disallow: /products?page=

# Allow Googlebot to crawl CSS/JS (required for rendering)
User-agent: Googlebot
Disallow:
```

Never disallow CSS and JS files — Googlebot must render your page to evaluate it.

---

## XML Sitemaps

### Basic Sitemap Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2024-09-01</lastmod>
  </url>
  <url>
    <loc>https://example.com/about</loc>
    <lastmod>2024-06-15</lastmod>
  </url>
</urlset>
```

**Tag guidance:**
- `<loc>` — required, must be a fully-qualified URL with protocol.
- `<lastmod>` — recommended. Use W3C datetime format (`YYYY-MM-DD` or `YYYY-MM-DDTHH:MM:SSZ`). Only update when content actually changes — Google validates this and demotes dishonest lastmod signals.
- `<changefreq>` and `<priority>` — **Google ignores both**. Omit them to reduce file size.

### Sitemap Index (for sites > 50,000 URLs or > 50MB)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/sitemap-pages.xml</loc>
    <lastmod>2024-09-01</lastmod>
  </sitemap>
  <sitemap>
    <loc>https://example.com/sitemap-products.xml</loc>
    <lastmod>2024-09-01</lastmod>
  </sitemap>
</sitemapindex>
```

### Next.js Dynamic Sitemap (App Router)

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next'

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const products = await fetch('https://api.example.com/products').then(r => r.json())

  const productUrls = products.map((p: { slug: string; updatedAt: string }) => ({
    url: `https://example.com/products/${p.slug}`,
    lastModified: new Date(p.updatedAt),
    changeFrequency: 'weekly' as const,
    priority: 0.8,
  }))

  return [
    { url: 'https://example.com', lastModified: new Date(), priority: 1 },
    { url: 'https://example.com/about', lastModified: new Date('2024-01-01'), priority: 0.5 },
    ...productUrls,
  ]
}
```

Submit sitemaps to Google Search Console via the Sitemaps report or reference them in `robots.txt`.

---

## Canonical URLs

### Purpose

Canonical URLs tell Google which URL is the "master" version when duplicate or near-duplicate content exists across multiple URLs. Consolidates link equity to a single URL.

### HTML Implementation

```html
<link rel="canonical" href="https://example.com/products/widget" />
```

Always use:
- **Absolute URLs** (including protocol and domain)
- **HTTPS** when the site supports it
- **No trailing slash inconsistency** — be consistent; canonical + sitemap + internal links should all agree

### When to Set Canonicals

| Scenario | Canonical target |
|---|---|
| Paginated series (`/blog?page=2`) | Each page self-canonicalizes — do NOT canonical page 2 to page 1 |
| Faceted navigation (`/shoes?color=red&size=10`) | Canonical to the base category `/shoes` |
| UTM-tracked URLs (`/page?utm_source=email`) | Canonical to clean URL `/page` |
| HTTP and HTTPS versions coexist | Canonical to HTTPS; also 301 redirect HTTP → HTTPS |
| `www` and non-`www` coexist | Canonical to preferred version; 301 redirect the other |
| Print versions (`/article?format=print`) | Canonical to the standard page |
| Cross-domain content syndication | Canonical on the syndicated copy pointing to the original |

### Canonical Gotchas

- Google treats canonicals as **hints, not directives** — if your canonical is inconsistent with other signals (sitemap, internal links, redirects), Google may override it.
- A canonical on a `noindex` page creates a contradiction — Google will usually ignore one of the signals.
- Self-referential canonicals (a page pointing to itself) are correct and recommended. They prevent Google from picking an alternative canonical.
- Next.js App Router: set via `alternates.canonical` in `metadata` or `generateMetadata`.

---

## hreflang — International SEO

### When to Use

Use `hreflang` when:
- The same site serves pages in multiple languages (e.g., `/en/about`, `/fr/about`)
- The same language serves multiple regions with different content (e.g., `en-US` vs `en-GB` pricing)

Do **not** use `hreflang` for:
- Pages that are identical in different languages (use canonical instead)
- Country-specific homepages with different domains pointing at completely different content — only use for pages that are translations/localizations of each other

### HTML Link Tag Method

Add to `<head>` of **every** page in the set. Every page must reference every other page, including itself:

```html
<!-- On https://example.com/en/about -->
<link rel="alternate" hreflang="en" href="https://example.com/en/about" />
<link rel="alternate" hreflang="fr" href="https://example.com/fr/about" />
<link rel="alternate" hreflang="de" href="https://example.com/de/about" />
<link rel="alternate" hreflang="x-default" href="https://example.com/about" />
```

`x-default` designates the fallback page for users whose language doesn't match any variant.

### HTTP Header Method (for PDFs and non-HTML files)

```
Link: <https://example.com/en/report.pdf>; rel="alternate"; hreflang="en",
      <https://example.com/fr/report.pdf>; rel="alternate"; hreflang="fr"
```

### XML Sitemap Method

```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://example.com/en/about</loc>
    <xhtml:link rel="alternate" hreflang="en" href="https://example.com/en/about"/>
    <xhtml:link rel="alternate" hreflang="fr" href="https://example.com/fr/about"/>
    <xhtml:link rel="alternate" hreflang="x-default" href="https://example.com/about"/>
  </url>
  <url>
    <loc>https://example.com/fr/about</loc>
    <xhtml:link rel="alternate" hreflang="en" href="https://example.com/en/about"/>
    <xhtml:link rel="alternate" hreflang="fr" href="https://example.com/fr/about"/>
    <xhtml:link rel="alternate" hreflang="x-default" href="https://example.com/about"/>
  </url>
</urlset>
```

### Next.js hreflang (App Router)

```tsx
// app/[lang]/about/page.tsx
export async function generateMetadata({ params }: { params: Promise<{ lang: string }> }): Promise<Metadata> {
  return {
    alternates: {
      canonical: `https://example.com/${(await params).lang}/about`,
      languages: {
        'en': 'https://example.com/en/about',
        'fr': 'https://example.com/fr/about',
        'de': 'https://example.com/de/about',
        'x-default': 'https://example.com/about',
      },
    },
  }
}
```

### Language Code Format

- Use **ISO 639-1** language codes: `en`, `fr`, `de`, `ja`, `ko`, `zh`
- Append **ISO 3166-1 alpha-2** region codes with a hyphen when needed: `en-US`, `en-GB`, `zh-TW`
- Invalid: region-only codes (`US`), unofficial codes (`uk` for Ukraine is valid, `UK` for United Kingdom is not — use `en-GB`)

### Common hreflang Mistakes

- Missing reciprocal links — if page A links to page B, page B **must** link back to page A
- Mixing language code formats (e.g. `en_US` with underscore instead of `en-US`)
- Using `hreflang` without a `canonical` — both should coexist; the canonical and hreflang self-reference should match
- Forgetting `x-default` on the language selector / fallback page

---

## Next.js Full Metadata API Examples

### Root Layout — Site-Wide Defaults

```tsx
// app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),    // makes relative URLs absolute
  title: {
    default: 'Acme',
    template: '%s | Acme',
  },
  description: 'Acme builds tools for modern teams.',
  openGraph: {
    siteName: 'Acme',
    locale: 'en_US',
    type: 'website',
    images: [{ url: '/og-default.png', width: 1200, height: 630 }],
  },
  twitter: {
    card: 'summary_large_image',
    creator: '@acme',
  },
  robots: {
    index: true,
    follow: true,
    googleBot: {
      index: true,
      follow: true,
      'max-image-preview': 'large',
      'max-snippet': -1,
    },
  },
  verification: {
    google: 'YOUR_GOOGLE_SITE_VERIFICATION_TOKEN',
  },
}
```

### Static Page Metadata

```tsx
// app/about/page.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'About',               // → "About | Acme" via template
  description: 'Learn about the Acme team and mission.',
  alternates: { canonical: '/about' },
}
```

### Dynamic Page Metadata with Data Fetching

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next'

type Props = { params: Promise<{ slug: string }> }

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const { slug } = await params
  const post = await fetch(`https://api.example.com/posts/${slug}`).then(r => r.json())

  // Merge with parent OG images rather than replace
  const parentImages = (await parent).openGraph?.images ?? []

  return {
    title: post.title,
    description: post.excerpt,
    alternates: { canonical: `/blog/${slug}` },
    openGraph: {
      type: 'article',
      publishedTime: post.publishedAt,
      modifiedTime: post.updatedAt,
      authors: [post.author.name],
      images: [{ url: post.coverImage, width: 1200, height: 630 }, ...parentImages],
    },
  }
}
```

### Robots Meta — Blocking Specific Pages

```tsx
// app/admin/layout.tsx  or  app/checkout/layout.tsx
export const metadata: Metadata = {
  robots: { index: false, follow: false },
}
```

---

## React / SPA: SSR vs SSG vs ISR Decision Criteria

### Rendering Mode Comparison

| Mode | How it works | SEO impact | When to use |
|---|---|---|---|
| **SSG** (Static Site Generation) | HTML pre-built at build time | Best — full HTML on first byte | Marketing pages, blogs, docs, product catalogs with infrequent updates |
| **SSR** (Server-Side Rendering) | HTML generated per request on the server | Good — full HTML, slower TTFB than SSG | Personalized pages, real-time data, pages that vary per user/session |
| **ISR** (Incremental Static Regeneration) | SSG with background revalidation on a timer | Good — mostly static TTFB | High-traffic pages that update periodically (news, pricing, inventory) |
| **CSR** (Client-Side Rendering) | Empty HTML shell; content rendered in browser JS | Risky — Googlebot can render JS but has limitations and delays | Dashboards, auth-gated tools, content Google should not index |

### Hydration Gotchas

- **Hydration mismatch**: Server HTML must match client React tree on first render. Common causes: `new Date()`, `Math.random()`, browser APIs (window, localStorage) called during SSR. Wrap these in `useEffect` or use `typeof window !== 'undefined'` guards.
- **Interactivity cliff**: A page may *look* loaded but be unresponsive until client JS hydrates. This tanks INP during load. Use React Server Components to ship less JS; defer non-critical client components.
- **CSR and crawlability**: Googlebot does eventually render JavaScript, but there is a crawl delay (can be days). For SEO-critical content, never rely solely on CSR.
- **SPA client-side navigation**: `document.title` updates via `history.pushState` — verify that title and canonical update correctly on navigation. Next.js App Router handles this automatically; vanilla SPAs must update manually.

---

## OpenGraph + Twitter Card

### OpenGraph — Full Tag Reference

```html
<!-- Required for all types -->
<meta property="og:title" content="Page Title — max ~60 chars" />
<meta property="og:description" content="Page description — aim for 2 sentences" />
<meta property="og:url" content="https://example.com/page" />
<meta property="og:image" content="https://example.com/og.png" />  <!-- absolute URL required -->
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta property="og:image:alt" content="Description of the image" />
<meta property="og:type" content="website" />  <!-- or: article, product, profile -->
<meta property="og:site_name" content="Acme" />
<meta property="og:locale" content="en_US" />

<!-- Article-specific -->
<meta property="article:published_time" content="2024-06-01T09:00:00Z" />
<meta property="article:modified_time" content="2024-09-15T12:00:00Z" />
<meta property="article:author" content="https://example.com/authors/jane" />
<meta property="article:section" content="Engineering" />
<meta property="article:tag" content="performance" />
```

**Image requirements:**
- Minimum 1200×630px for `summary_large_image` display
- PNG or JPG; JPEG preferred for photos (smaller files)
- Max 8MB; images > 5MB may not render on some platforms
- Ratio must be between 1.91:1 and 1:1; 1.91:1 (1200×630) is standard

### Twitter (X) Card — Full Tag Reference

```html
<meta name="twitter:card" content="summary_large_image" />  <!-- summary | summary_large_image | app | player -->
<meta name="twitter:site" content="@acme" />                <!-- site's Twitter handle -->
<meta name="twitter:creator" content="@janedoe" />          <!-- author's handle -->
<meta name="twitter:title" content="Page Title" />
<meta name="twitter:description" content="Description up to 200 chars" />
<meta name="twitter:image" content="https://example.com/og.png" />
<meta name="twitter:image:alt" content="Image description" />
```

Twitter falls back to OpenGraph tags if Twitter-specific tags are absent. Always set `twitter:card` explicitly — without it, Twitter defaults to `summary` (small square thumbnail).

---

## Semantic HTML for SEO

### Heading Hierarchy Rules

One `<h1>` per page. It should contain the primary keyword and match (or closely relate to) the `<title>` tag.

```html
<h1>Core Web Vitals Optimization Guide</h1>
  <h2>Understanding LCP</h2>
    <h3>What counts as an LCP element</h3>
    <h3>Fixing slow LCP</h3>
  <h2>Understanding INP</h2>
    <h3>Measuring interactions</h3>
```

Never skip levels (e.g. `<h1>` → `<h3>`). Heading levels communicate document outline to crawlers and screen readers alike.

### Landmark Elements

```html
<header>      <!-- site-wide header, logo, primary nav -->
  <nav>       <!-- primary navigation — use aria-label="Primary" if multiple navs -->
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/products">Products</a></li>
    </ul>
  </nav>
</header>

<main>        <!-- one per page; the unique, primary content -->
  <article>   <!-- self-contained content: blog post, news item, product card -->
    <h1>Article Title</h1>
    <p>...</p>
  </article>
  <aside>     <!-- supplementary content: related links, ads, author bio -->
    ...
  </aside>
</main>

<footer>      <!-- site-wide footer -->
  <nav aria-label="Footer">...</nav>
</footer>
```

Google uses landmark elements to identify the primary content region. Content inside `<main>` is weighted more heavily than content in `<aside>` or `<footer>`.

### `alt` Text Rules

| Image type | Correct `alt` |
|---|---|
| Informative (photo of product) | Descriptive text: `alt="Acme Widget in matte black, front view"` |
| Decorative (divider, background) | Empty string: `alt=""` — tells screen readers to skip it |
| Functional (button icon, logo) | Describe the action/destination: `alt="Acme home"` |
| Complex (chart, graph) | Short alt + long description in adjacent text or `longdesc` |

Formula for informative images: **[Subject] [action/state] [relevant context]**
Example: `alt="Engineer reviewing a dashboard on a laptop"` — not `alt="photo"` or `alt="image1.jpg"`.

Missing `alt` is a Lighthouse SEO audit failure and an accessibility violation (WCAG 1.1.1).

### Additional Semantic Elements

```html
<time datetime="2024-06-01">June 1, 2024</time>   <!-- machine-readable dates -->
<address>                                           <!-- contact info for nearest article/body -->
  Written by <a href="/authors/jane">Jane Doe</a>
</address>
<figure>
  <img src="chart.png" alt="Bar chart showing 40% CLS improvement" />
  <figcaption>CLS scores before and after optimization</figcaption>
</figure>
<abbr title="Cumulative Layout Shift">CLS</abbr>   <!-- first use of abbreviations -->
```
