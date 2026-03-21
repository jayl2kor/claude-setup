# Playwright Patterns Reference

Detailed implementation patterns for Playwright. Covers the full Page Object Model class, composable fixtures with `test.extend()`, network mocking, visual regression, accessibility checks, and flaky test management.

---

## Full Page Object Model: CheckoutPage

```typescript
// tests/pages/CheckoutPage.ts
import { type Page, type Locator, expect } from '@playwright/test'

export class CheckoutPage {
  private readonly page: Page
  private readonly emailInput: Locator
  private readonly passwordInput: Locator
  private readonly submitButton: Locator
  private readonly errorBanner: Locator
  readonly orderConfirmationHeading: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel('Email address')
    this.passwordInput = page.getByLabel('Password')
    this.submitButton = page.getByRole('button', { name: 'Place order' })
    this.errorBanner = page.getByRole('alert')
    this.orderConfirmationHeading = page.getByRole('heading', { name: /order confirmed/i })
  }

  async goto() {
    await this.page.goto('/checkout')
  }

  async fillShipping(details: { email: string; address: string; city: string }) {
    await this.emailInput.fill(details.email)
    await this.page.getByLabel('Street address').fill(details.address)
    await this.page.getByLabel('City').fill(details.city)
  }

  async placeOrder() {
    await this.submitButton.click()
    // Wait for navigation OR confirmation element — do not use waitForTimeout
    await this.orderConfirmationHeading.waitFor({ state: 'visible', timeout: 15_000 })
  }

  async expectError(message: string | RegExp) {
    await expect(this.errorBanner).toBeVisible()
    await expect(this.errorBanner).toContainText(message)
  }
}
```

---

## Composable Fixtures with `test.extend()`

Use `test.extend()` to compose reusable authenticated contexts instead of repeating login logic in every test.

```typescript
// tests/fixtures/auth.ts
import { test as base, type Page } from '@playwright/test'
import { CheckoutPage } from '../pages/CheckoutPage'

type Fixtures = {
  authenticatedPage: Page
  checkoutPage: CheckoutPage
}

export const test = base.extend<Fixtures>({
  authenticatedPage: async ({ page, request }, use) => {
    // Inject auth cookie directly via API — avoid UI login on every test
    const response = await request.post('/api/auth/test-login', {
      data: { email: 'test@example.com', role: 'customer' },
    })
    const { token } = await response.json()
    await page.context().addCookies([
      { name: 'session', value: token, domain: 'localhost', path: '/' },
    ])
    await use(page)
  },

  checkoutPage: async ({ authenticatedPage }, use) => {
    const co = new CheckoutPage(authenticatedPage)
    await co.goto()
    await use(co)
  },
})

export { expect } from '@playwright/test'
```

```typescript
// tests/e2e/checkout.spec.ts
import { test, expect } from '../fixtures/auth'

test('completes purchase for valid order', async ({ checkoutPage }) => {
  await checkoutPage.fillShipping({
    email: 'buyer@example.com',
    address: '123 Main St',
    city: 'Springfield',
  })
  await checkoutPage.placeOrder()
  await expect(checkoutPage.orderConfirmationHeading).toBeVisible()
})
```

---

## Network Mocking with `page.route()`

### Mock a specific endpoint to control response deterministically

```typescript
test('shows error banner when inventory API fails', async ({ page }) => {
  await page.route('**/api/inventory/**', route =>
    route.fulfill({ status: 503, body: JSON.stringify({ error: 'Service unavailable' }) })
  )

  await page.goto('/products/widget-42')
  await expect(page.getByRole('alert')).toContainText(/unavailable/i)
})
```

### Spy on requests without mocking

```typescript
test('sends correct payload on order submit', async ({ page }) => {
  const [request] = await Promise.all([
    page.waitForRequest(r => r.url().includes('/api/orders') && r.method() === 'POST'),
    page.getByRole('button', { name: 'Place order' }).click(),
  ])

  const body = request.postDataJSON()
  expect(body.items).toHaveLength(2)
  expect(body.currency).toBe('USD')
})
```

### Abort requests selectively (e.g., block analytics in tests)

```typescript
test.beforeEach(async ({ page }) => {
  await page.route('**/analytics/**', route => route.abort())
  await page.route('**/hotjar.com/**', route => route.abort())
})
```

### HAR recording and playback

```typescript
// Record: run once to capture real responses
await page.routeFromHAR('fixtures/checkout.har', { update: true })

// Playback: use recorded responses deterministically in CI
await page.routeFromHAR('fixtures/checkout.har', { update: false })
```

---

## Visual Regression Testing

Use Playwright's built-in snapshot assertions for pixel-level comparison.

```typescript
// Use built-in Playwright snapshots for pixel-level comparison
test('product card renders correctly', async ({ page }) => {
  await page.goto('/products/widget-42')
  await page.waitForLoadState('networkidle')

  // Mask dynamic content (timestamps, user-specific data)
  await expect(page.getByTestId('product-card')).toHaveScreenshot('product-card.png', {
    mask: [page.getByTestId('last-updated'), page.getByTestId('live-price')],
    maxDiffPixelRatio: 0.02,  // Allow 2% pixel difference
  })
})

// Full page snapshot
test('checkout page layout', async ({ page }) => {
  await page.goto('/checkout')
  await expect(page).toHaveScreenshot('checkout-full.png', { fullPage: true })
})
```

Update baselines intentionally:

```bash
npx playwright test --update-snapshots
```

**Guidelines for stable visual tests:**
- Always call `waitForLoadState('networkidle')` before taking screenshots
- Mask all dynamic content: timestamps, prices, user avatars, ad slots
- Set `maxDiffPixelRatio` to accommodate sub-pixel anti-aliasing differences across OSes
- Commit baseline `.png` files to version control; update them in dedicated PRs

---

## Accessibility Checks with axe-playwright

```typescript
import { checkA11y, injectAxe } from 'axe-playwright'

test('checkout page has no critical a11y violations', async ({ page }) => {
  await page.goto('/checkout')
  await injectAxe(page)
  await checkA11y(page, undefined, {
    runOnly: { type: 'tag', values: ['wcag2a', 'wcag2aa'] },
    detailedReport: true,
  })
})
```

**Scope checks to a component instead of the full page:**

```typescript
await checkA11y(page, '#product-card', {
  runOnly: { type: 'tag', values: ['wcag2a', 'wcag2aa'] },
})
```

**Exclude known violations while tracking them:**

```typescript
await checkA11y(page, undefined, {
  runOnly: { type: 'tag', values: ['wcag2aa'] },
  rules: {
    'color-contrast': { enabled: false },  // tracked in #789, fix in next sprint
  },
})
```

---

## Flaky Test Handling

### Quarantine without deleting

```typescript
// Tag a flaky test without deleting it
test('flaky: payment modal animation', { tag: '@flaky' }, async ({ page }) => {
  test.fixme(true, 'Intermittent — tracked in issue #456')
  // ...
})

// Run only stable tests in CI gate; run flaky set separately
// npx playwright test --grep-invert @flaky
// npx playwright test --grep @flaky --retries=5
```

### Identify flakiness before merging

```bash
npx playwright test tests/checkout.spec.ts --repeat-each=20 --retries=0
```

### Common causes and fixes

| Cause | Symptom | Fix |
|---|---|---|
| Arbitrary `waitForTimeout` | Passes locally, fails in slow CI | Replace with `waitForResponse` or `expect(locator).toBeVisible()` |
| Shared test state | Test passes alone, fails in suite | Ensure full data isolation per test |
| Animation timing | Screenshot diff on fast machines | `await page.waitForLoadState('networkidle')` + mask animated elements |
| Race between navigation and assertion | `Page closed` or `Detached` errors | Use `page.waitForURL()` before asserting |
| Missing `await` on assertion | Always passes (assertion never runs) | TypeScript strict mode + ESLint `no-floating-promises` |

### Retry strategy in `playwright.config.ts`

```typescript
// Retry on CI; never locally (masks real failures)
retries: process.env.CI ? 2 : 0,
```

After a retry succeeds, Playwright marks the test as "flaky" in the report — use this signal to identify tests needing attention.
