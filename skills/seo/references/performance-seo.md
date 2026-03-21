# Performance SEO Reference

Deep-dive on Core Web Vitals (LCP, INP, CLS) with specific fixes, Lighthouse SEO audit interpretation, and Google Search Console monitoring. See [../SKILL.md](../SKILL.md) for thresholds and the quick-reference summary.

---

## LCP — Largest Contentful Paint

### What Counts as the LCP Element

The browser tracks which of the following is the largest visible element at each point during page load:

- `<img>` elements (including `srcset` and `<picture>` sources)
- `<image>` elements inside SVG
- `<video>` elements — the poster image counts, or the time the first frame is presented if no poster
- Block-level elements with `background-image: url(...)` loaded via CSS
- Block-level elements containing text (the text block, not the font file)

The LCP element can change as the page loads — if a hero image loads after an initially large text block, the hero image becomes the LCP element. Google records the final LCP element at load completion.

**Elements excluded from LCP candidacy:**
- Elements with `opacity: 0` or `visibility: hidden`
- Elements covering the full viewport (treated as background)
- Low-entropy placeholder images (solid-color blocks)

### LCP Fix Hierarchy

Address these in order — earlier steps give the biggest gains:

**1. Eliminate render-blocking resources above the fold**

```html
<!-- BAD: synchronous stylesheet blocks rendering -->
<link rel="stylesheet" href="/styles/global.css" />

<!-- GOOD: inline critical CSS; async-load the rest -->
<style>/* above-fold critical styles inlined */</style>
<link rel="preload" href="/styles/global.css" as="style" onload="this.onload=null;this.rel='stylesheet'" />
```

**2. Set `fetchpriority="high"` on the LCP image**

```html
<!-- Tells the browser to fetch this before other images — biggest single LCP win -->
<img src="/hero.webp" fetchpriority="high" alt="Hero image" width="1200" height="630" />
```

In Next.js, the `<Image>` component `priority` prop does this automatically:

```tsx
import Image from 'next/image'
<Image src="/hero.webp" priority alt="Hero" width={1200} height={630} />
```

**3. Preload LCP images referenced in CSS**

If the LCP image is a CSS `background-image` or loaded by JavaScript, the browser cannot discover it until CSS/JS parses. Force early discovery:

```html
<link rel="preload" fetchpriority="high" as="image"
      href="/hero.webp"
      imagesrcset="/hero-400.webp 400w, /hero-800.webp 800w, /hero.webp 1200w"
      imagesizes="100vw" />
```

**4. Reduce TTFB**

Nothing renders until the server delivers the first byte. Target < 800ms TTFB:
- Use a CDN to serve cached HTML from edge nodes geographically close to users
- Enable HTTP/2 or HTTP/3
- Eliminate unnecessary redirects (each adds 100–300ms)
- Use SSG or ISR for pages that don't require per-request server rendering
- Avoid URL parameters that bust CDN cache (`?utm_*` should be stripped at the CDN before cache lookup, not after)

**5. Use modern image formats and responsive sizing**

```html
<picture>
  <source type="image/avif" srcset="/hero.avif 1x, /hero-2x.avif 2x" />
  <source type="image/webp" srcset="/hero.webp 1x, /hero-2x.webp 2x" />
  <img src="/hero.jpg" alt="Hero" width="1200" height="630" fetchpriority="high" />
</picture>
```

AVIF is typically 30–50% smaller than WebP; WebP is 25–35% smaller than JPEG.

**6. Server response optimization**

- Enable compression (Brotli > gzip)
- Cache HTML at CDN for SSG pages; use stale-while-revalidate headers
- Move time-consuming database queries to background jobs or edge compute

---

## INP — Interaction to Next Paint

INP replaced FID (First Input Delay) as a Core Web Vital in March 2024. INP measures the **worst-case interaction latency** across the entire page session, not just the first interaction.

### How INP is Measured

An interaction consists of three phases:

```
User input
    ↓
[Input delay]       ← time until event handlers start running
    ↓
[Processing time]   ← time event handlers take to execute
    ↓
[Presentation delay]← time browser takes to paint the next frame after handlers complete
    ↓
Frame presented to user
```

INP = sum of all three phases for the slowest interaction.

### Diagnosing Slow INP

Use the `web-vitals` library to capture interaction attribution in the field:

```ts
import { onINP } from 'web-vitals/attribution'

onINP(({ value, attribution }) => {
  console.log('INP:', value, 'ms')
  console.log('Slow interaction type:', attribution.interactionType)   // click, keydown, pointer
  console.log('Long animation frame:', attribution.longAnimationFrameEntries)
})
```

In Chrome DevTools, use the Performance panel's "Interactions" track to identify which specific clicks or keystrokes are slow.

### INP Fix Patterns

**Break up long event handlers**

Any synchronous work over ~50ms blocks the main thread and adds to INP. Split into separate tasks:

```ts
// BAD: single 200ms synchronous handler
button.addEventListener('click', () => {
  updateUI()          // 30ms
  runExpensiveCalc()  // 120ms
  logAnalytics()      // 50ms
})

// GOOD: yield between UI update and non-critical work
button.addEventListener('click', () => {
  updateUI()          // runs immediately, unblocks paint
  setTimeout(() => {
    runExpensiveCalc()
    logAnalytics()
  }, 0)               // deferred to next task
})
```

**Yield to the main thread before non-rendering work**

```ts
async function handleClick() {
  updateUI()                    // critical — user sees this immediately
  await yieldToMain()           // allow browser to paint before continuing
  runExpensiveCalc()            // non-critical — runs after frame is rendered
}

function yieldToMain(): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, 0))
}
```

**Reduce DOM size**

Large DOMs (> 1,500 nodes) slow every layout, style recalculation, and paint. Lazy-render off-screen content:

```css
/* Defer layout/paint for off-screen elements */
.below-fold-section {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px;  /* estimated height prevents CLS */
}
```

**Avoid layout thrashing**

Reading layout properties after writing styles forces a synchronous layout:

```ts
// BAD: forces two synchronous layouts
element.style.width = '500px'
const height = element.offsetHeight  // forces layout recalculation
element.style.height = height + 'px'

// GOOD: batch reads before writes
const height = element.offsetHeight   // read first
element.style.width = '500px'         // write after all reads
element.style.height = height + 'px'
```

**Reduce input delay during page load**

The main thread is busiest during initial load. Long tasks (> 50ms) during this period create input delay for early interactions. In Next.js, defer non-critical JS with dynamic imports:

```tsx
import dynamic from 'next/dynamic'
const HeavyWidget = dynamic(() => import('./HeavyWidget'), { ssr: false })
```

---

## CLS — Cumulative Layout Shift

CLS measures the sum of all unexpected layout shifts, weighted by the shift distance and fraction of viewport affected. A score of 0 means nothing moved unexpectedly.

### Always Set Image Dimensions

The single most impactful CLS fix. The browser reserves no space for an image before it loads unless dimensions are specified:

```html
<!-- BAD: browser doesn't know size until image loads → layout shift when it arrives -->
<img src="product.jpg" alt="Product" />

<!-- GOOD: browser reserves the exact space immediately -->
<img src="product.jpg" alt="Product" width="640" height="480" />
```

Pair with CSS to allow responsive scaling while preserving aspect ratio:

```css
img {
  width: 100%;
  height: auto;   /* browser uses width/height attributes to compute aspect ratio */
}
```

For dynamically sized containers, use CSS `aspect-ratio`:

```css
.hero-image-container {
  aspect-ratio: 16 / 9;
  width: 100%;
}
```

### Fix Font-Induced CLS

Web fonts swap in and replace the fallback font, causing a layout shift if the fonts differ in metrics.

```css
/* Option 1: font-display: optional — no swap if font isn't cached, no CLS */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: optional;
}

/* Option 2: font-display: swap + metric overrides — swap but minimize shift */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap;
  size-adjust: 105%;          /* scale fallback to match web font metrics */
  ascent-override: 90%;       /* match ascent of web font */
  descent-override: 20%;
  line-gap-override: 0%;
}
```

Preloading the font file reduces the window during which the fallback is shown:

```html
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
```

### Dynamic Content CLS

Content inserted above the fold — ads, banners, cookie notices — causes the worst CLS because the entire page shifts down.

```css
/* Reserve space for an ad slot before the ad loads */
.ad-container {
  min-height: 250px;          /* typical ad height */
  width: 100%;
  background-color: #f5f5f5;  /* placeholder visual */
}
```

For cookie consent banners and notification bars: use `position: sticky` or `position: fixed` so they overlay content rather than push it down.

Avoid inserting elements above existing content programmatically:

```ts
// BAD: shifts all existing content down
document.body.insertAdjacentHTML('afterbegin', '<div class="announcement-bar">...</div>')

// GOOD: space reserved in initial HTML; JS only toggles visibility
document.querySelector('.announcement-bar').style.display = 'block'
```

### Animations and Transitions

CSS transitions that change `top`, `left`, `width`, `height`, or `margin` trigger layout and contribute to CLS. Use `transform` instead:

```css
/* BAD: changes layout geometry, counted as a layout shift */
.panel { transition: height 0.3s ease; }

/* GOOD: GPU-composited transform, never counted as CLS */
.panel { transition: transform 0.3s ease; }
.panel.open { transform: translateY(0); }
.panel.closed { transform: translateY(-100%); }
```

---

## Lighthouse SEO Audit Interpretation

Run `lighthouse https://example.com --only-categories=seo --output=json` or use Chrome DevTools Lighthouse tab.

### Audit Failures and Fixes

| Audit | What it checks | Common fix |
|---|---|---|
| **Document does not have a meta description** | `<meta name="description">` present and non-empty | Add description, 150–160 chars |
| **Document title element is not descriptive** | `<title>` is not empty or generic | Write unique, keyword-relevant title per page |
| **Page is blocked from indexing** | `noindex` in meta robots or X-Robots-Tag | Remove `noindex` from production; check staging env vars |
| **Links are not crawlable** | `<a>` tags have no `href` or use `href="#"` | Use real `href` values; ensure JS-rendered links have `href` in SSR HTML |
| **Image elements do not have [alt] attributes** | `<img>` missing `alt` | Add descriptive `alt`; `alt=""` for decorative images |
| **Robots.txt is not valid** | Syntax errors or inaccessible robots.txt | Validate with Google's robots.txt Tester |
| **Document doesn't have a valid hreflang** | Missing reciprocal links or invalid language codes | Audit all hreflang pairs; fix ISO codes |
| **Document does not have a valid rel=canonical** | Canonical URL not found or returns non-200 | Ensure canonical URL is live and returns 200 |
| **Structured data is invalid** | JSON-LD fails schema.org validation | Run Rich Results Test; fix property types and required fields |
| **Tap targets are not sized appropriately** | Mobile touch targets < 48×48px | Increase button/link padding; minimum 48px touch target |

Lighthouse SEO score is diagnostic, not a ranking signal. A score of 100 means no audit failures detected — it does not mean good rankings.

### Lighthouse vs Field Data

Lighthouse tests a single page load from a single location with no cache. Core Web Vitals in Google Search Console use **real user data (CrUX)** at the 75th percentile. A page can pass Lighthouse but still fail CWV in Search Console (because real users have slower devices, worse networks, or the page behaves differently under load).

Always validate CWV using Search Console or a RUM solution, not just Lighthouse.

---

## Google Search Console — Key Metrics to Watch

### Performance Report

The primary report for organic search health. Segment by **Search type: Web** and use a 28-day window for stable data.

| Metric | What it means | When to investigate |
|---|---|---|
| **Total clicks** | Users who clicked your site from Google | Sudden drops > 20% week-over-week |
| **Total impressions** | Times your site appeared in search results | Drop without click drop = CTR problem; impression drop = indexing/ranking problem |
| **Average CTR** | Clicks ÷ Impressions | CTR below 2% for branded queries; below 1% for competitive queries |
| **Average position** | Mean ranking position across all queries | Position > 20 for high-value queries = content/authority problem |

**Query analysis:** Filter for queries with high impressions but low CTR — these pages rank but have unappealing titles or descriptions. Rewrite the title tag and meta description.

**Page analysis:** Identify pages with declining impressions — either they were deindexed, lost ranking, or a canonical changed.

### Index Coverage Report

| Status | Meaning | Action |
|---|---|---|
| **Valid** | Indexed and eligible to rank | Monitor for unexpected drops |
| **Valid with warnings** | Indexed but issues detected (e.g. submitted URL not canonical) | Fix canonical/sitemap mismatch |
| **Excluded — Crawled, not indexed** | Google crawled but decided not to index | Improve content quality; check for thin content or near-duplicates |
| **Excluded — Discovered, not crawled** | URL known but not yet fetched | Add to sitemap; increase internal linking to signal priority |
| **Error — Redirect error** | 3xx chains or loops | Fix redirect chains to single hop |
| **Error — Not found (404)** | 404 for a submitted URL | Fix broken URL or remove from sitemap |
| **Excluded — Blocked by robots.txt** | Disallowed in robots.txt | Intentional? If not, update robots.txt |
| **Excluded — Noindex** | `noindex` directive found | Intentional? If not, remove noindex |

### Core Web Vitals Report

Shows real-user CWV data from Chrome User Experience Report (CrUX). Requires sufficient traffic to generate data (typically a few hundred page views per day per URL group).

- URLs are grouped by similarity — fixing one URL fixes the whole URL pattern's score
- "Poor" URLs directly impact Page Experience ranking signal
- Mobile and desktop are reported separately — mobile failures are more impactful (Google uses mobile-first indexing)
- After fixing an issue, use "Validate fix" in Search Console. Re-classification takes 28 days of new data

### URL Inspection Tool

Use to diagnose individual URLs:

1. **Last crawl date** — if very old (> 3 months), increase internal links to the page or submit URL directly
2. **Indexing status** — confirms whether the page is indexed
3. **Canonical URL** — shows which URL Google selected as canonical (may differ from your declared canonical)
4. **Enhancements** — shows whether structured data was parsed and if it has errors
5. **Test Live URL** — renders the page with Googlebot and shows what HTML is actually visible (critical for debugging CSR/SSR issues)

### Manual Actions and Security Issues

Check these monthly:
- **Manual Actions**: Google quality rater found a policy violation — requires fixing the issue and submitting a reconsideration request
- **Security Issues**: Hacked content, malware, social engineering — must be resolved before rankings recover; submit request after cleanup
