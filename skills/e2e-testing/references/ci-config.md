# CI Configuration Reference

Full configuration files for Playwright in CI/CD. Covers `playwright.config.ts` with sharding, the GitHub Actions workflow with shard matrix and merged reports, and retry strategies.

---

## Full `playwright.config.ts` with Sharding

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 4 : undefined,
  timeout: 30_000,
  expect: { timeout: 8_000 },
  reporter: [
    ['html', { outputFolder: 'playwright-report', open: 'never' }],
    ['junit', { outputFile: 'results/junit.xml' }],
    process.env.CI ? ['github'] : ['list'],
  ],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit',  use: { ...devices['Desktop Safari'] } },
    { name: 'mobile',  use: { ...devices['Pixel 7'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
})
```

### Key config decisions

| Option | CI value | Local value | Rationale |
|---|---|---|---|
| `forbidOnly` | `true` | `false` | Prevents accidentally committing `test.only` |
| `retries` | `2` | `0` | Masks flakiness locally; CI retries are acceptable |
| `workers` | `4` | `undefined` (auto) | Fixed parallelism for predictable shard sizes |
| `trace` | `on-first-retry` | `on-first-retry` | Captures trace only when needed, not every run |
| `video` | `retain-on-failure` | `retain-on-failure` | Available for debugging without bloating artifacts |

---

## GitHub Actions CI Workflow with Shard Matrix

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on:
  push:
    branches: [main]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]   # 4-way parallelism across separate runners
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npx playwright install --with-deps chromium firefox
      - name: Run shard
        run: npx playwright test --shard=${{ matrix.shard }}/4
        env:
          BASE_URL: ${{ vars.STAGING_URL }}
          CI: true
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report-shard-${{ matrix.shard }}
          path: playwright-report/
          retention-days: 14

  merge-reports:
    needs: e2e
    if: always()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { path: all-reports, pattern: playwright-report-* }
      - run: npx playwright merge-reports --reporter html ./all-reports
      - uses: actions/upload-artifact@v4
        with:
          name: merged-playwright-report
          path: playwright-report/
```

### Sharding strategy

- **4 shards** is a good starting point for suites up to ~300 tests
- Increase to 8 shards for suites over 500 tests or when wall-clock time exceeds 10 minutes
- `fail-fast: false` ensures all shards complete so the merged report is always full
- `if: always()` on `merge-reports` guarantees a report even when some shards fail

### Caching Playwright browsers

Add this step before `npx playwright install` to cache browser binaries across runs:

```yaml
- name: Cache Playwright browsers
  uses: actions/cache@v4
  id: playwright-cache
  with:
    path: ~/.cache/ms-playwright
    key: playwright-${{ runner.os }}-${{ hashFiles('package-lock.json') }}

- name: Install Playwright browsers
  if: steps.playwright-cache.outputs.cache-hit != 'true'
  run: npx playwright install --with-deps chromium firefox
```

---

## Retry Strategies

### In-process retries (`playwright.config.ts`)

```typescript
retries: process.env.CI ? 2 : 0,
```

Playwright retries the full test on failure. On success after a retry, the test is marked "flaky" in the report — use this as a signal to investigate.

### Re-run only failed tests (fast feedback loop)

After a CI run with failures, re-run only the failed tests to confirm fixes:

```bash
npx playwright test --last-failed
```

### Quarantine flaky tests from the main CI gate

```bash
# Main CI gate — skip all @flaky tagged tests
npx playwright test --grep-invert @flaky

# Nightly or separate job — run only @flaky tests with high retry count
npx playwright test --grep @flaky --retries=5
```

### Detect flakiness locally before pushing

```bash
# Run the same test 20 times with no retries — any failure reveals flakiness
npx playwright test tests/checkout.spec.ts --repeat-each=20 --retries=0
```

### Soft assertions for non-blocking checks

```typescript
// Does not stop the test on failure — collects all failures first
test('dashboard smoke', async ({ page }) => {
  await page.goto('/dashboard')
  await expect.soft(page.getByTestId('revenue-chart')).toBeVisible()
  await expect.soft(page.getByTestId('user-count')).toBeVisible()
  // Test only fails at the end if any soft assertion failed
})
```
