---
name: e2e-testing
description: This skill should be used when the user is writing, reviewing, or debugging end-to-end tests using Playwright or Cypress; setting up test infrastructure; applying the Page Object Model; managing test data; integrating E2E tests into CI/CD pipelines; or diagnosing flaky tests and selector fragility.
version: 1.0.0
---

# E2E Testing Patterns

Expert Playwright patterns for building stable, fast, and maintainable E2E test suites. Covers the testing pyramid, selector strategies, POM, test data, CI/CD, visual regression, and antipatterns.

## When to Activate

- Writing new Playwright or Cypress E2E tests
- Diagnosing flaky or slow test suites
- Reviewing test code for fragile selectors or bad isolation
- Setting up CI parallelization or retry strategies
- Implementing the Page Object Model
- Adding visual regression coverage
- Choosing between Playwright and Cypress for a new project

---

## Testing Pyramid: Where E2E Fits

```
        /\
       /E2E\          ~5-10%   Slow, brittle — test critical user journeys only
      /------\
     /Integr. \       ~20-30%  API contracts, DB interactions, auth flows
    /----------\
   /    Unit    \     ~60-70%  Pure logic, transformations, validators
  /--------------\
```

**Rule of thumb**: Every bug found in E2E should prompt a question — "Can a unit or integration test catch this faster?" If yes, add one there too and keep the E2E test for the happy path only.

E2E tests are warranted for:
- Critical user journeys (signup → purchase → confirmation)
- Cross-browser rendering correctness
- Auth state and cookie handling
- Third-party OAuth flows

---

## Playwright vs Cypress: When to Choose Which

| Criterion | Playwright | Cypress |
|---|---|---|
| Multi-browser (Safari, Firefox) | Native support | Firefox only; Safari via workarounds |
| Multi-tab / iframes | First-class | Limited |
| API mocking | `route()` built-in | `cy.intercept()` |
| Auto-waiting | Built-in on all actions | Built-in |
| Network replay / HAR | Yes | No |
| Component testing | Experimental | Mature |
| Speed at scale | Faster in CI (parallel workers) | Comparable for small suites |

**Choose Playwright** for new projects, multi-browser requirements, or complex network scenarios.
**Choose Cypress** if the team already has significant Cypress investment or needs mature component testing.

---

## Locator Strategy: Priority Order

Always prefer locators that survive implementation changes. Use this priority, highest to lowest:

```typescript
// 1. ARIA roles — most resilient, accessible by default
page.getByRole('button', { name: 'Submit order' })
page.getByRole('textbox', { name: 'Email address' })
page.getByRole('heading', { name: /dashboard/i })

// 2. data-testid — explicit contract between test and markup
page.getByTestId('order-summary-total')

// 3. Label text — good for form fields
page.getByLabel('Password')

// 4. Placeholder — acceptable for inputs without labels
page.getByPlaceholder('Search products…')

// 5. Text content — only for stable, non-i18n strings
page.getByText('No results found')

// AVOID — fragile, implementation-coupled
page.locator('.btn-primary > span:nth-child(2)')  // CSS path
page.locator('#app > div:first-child')            // DOM position
page.locator('//div[@class="modal"]/button')      // XPath anti-pattern
```

---

## Page Object Model

Structure each page as a class with locators declared in the constructor and actions as `async` methods. Never expose raw locators outside the POM class.

```typescript
// tests/pages/CheckoutPage.ts
import { type Page, type Locator, expect } from '@playwright/test'

export class CheckoutPage {
  private readonly page: Page
  private readonly emailInput: Locator
  private readonly submitButton: Locator
  readonly orderConfirmationHeading: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel('Email address')
    this.submitButton = page.getByRole('button', { name: 'Place order' })
    this.orderConfirmationHeading = page.getByRole('heading', { name: /order confirmed/i })
  }

  async goto() { await this.page.goto('/checkout') }

  async placeOrder() {
    await this.submitButton.click()
    await this.orderConfirmationHeading.waitFor({ state: 'visible', timeout: 15_000 })
  }
}
```

See [`references/playwright-patterns.md`](references/playwright-patterns.md) for the full POM class, composable fixtures with `test.extend()`, network mocking, visual regression, accessibility checks, and flaky test management.

---

## Test Data Strategies

Choose a strategy based on data complexity and isolation needs:

| Strategy | Best For | Isolation |
|---|---|---|
| **API seeding** (`beforeEach` POST) | Integration-heavy tests, DB state | High — reset after each test |
| **Fixtures JSON** | Static reference data, lookup tables | N/A — read-only |
| **Factory functions** | Dynamic entities with unique IDs | High — unique per test run |

Use factory functions to avoid ID collisions across parallel workers:

```typescript
export function buildUser(overrides: Partial<User> = {}): User {
  return {
    id: crypto.randomUUID(),
    email: `user+${Date.now()}@test.example`,
    role: 'customer',
    ...overrides,
  }
}
```

---

## Antipatterns: Quick Reference

| Antipattern | Why It Hurts | Fix |
|---|---|---|
| `waitForTimeout(3000)` | Race condition — passes on fast CI, fails on slow | `waitForResponse()` or `expect(locator).toBeVisible()` |
| CSS path selectors | Breaks on any DOM restructure | `getByRole()` or `getByTestId()` |
| Test B depends on Test A's data | Ordering dependency, parallel failures | Each test seeds its own data |
| `isVisible()` + `expect(bool)` | Bypasses auto-wait | `expect(locator).toBeVisible()` directly |
| E2E test for pure calculation logic | Slow feedback cycle | Unit test the logic; E2E only for happy path |
| No quarantine for known-flaky tests | Breaks CI for everyone | Tag `@flaky`, exclude from main gate, run separately |

---

## References

- [`references/playwright-patterns.md`](references/playwright-patterns.md) — Full POM TypeScript class, composable fixtures, network mocking, visual regression, axe-playwright, flaky test handling
- [`references/ci-config.md`](references/ci-config.md) — Full `playwright.config.ts` with sharding, GitHub Actions CI workflow with shard matrix and merge-reports, retry strategies
