---
name: e2e-testing
description: "E2E testing with Playwright: test structure, page objects, authentication, visual testing, CI configuration. Use when writing or improving E2E tests for web applications."
---

## E2E Testing with Playwright

E2E tests simulate real user journeys through the browser. Keep the suite small and focused on critical paths.

### Context

User journey or E2E testing problem: **$ARGUMENTS**

---

### What to E2E Test

```
Test:          - User registration and login flow
               - Checkout / payment flow
               - Core CRUD happy paths
               - Critical business workflows (booking, onboarding)

Don't test:    - Error messages (unit/integration handles this)
               - Every UI state (RTL handles this)
               - Admin-only edge cases
               - Things that need fake data manipulation

Target: 5-15 tests covering the most critical user journeys
```

---

### Playwright Setup

```javascript
// playwright.config.js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,  // fail if test.only committed
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html'], ['list']],

  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:5173',
    trace: 'on-first-retry',     // record trace on failure
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile', use: { ...devices['iPhone 14'] } },  // mobile viewport
  ],

  // Start dev servers before tests
  webServer: [
    {
      command: 'npm run dev:backend',
      url: 'http://localhost:3000/health',
      reuseExistingServer: !process.env.CI,
    },
    {
      command: 'npm run dev:frontend',
      url: 'http://localhost:5173',
      reuseExistingServer: !process.env.CI,
    },
  ],
});
```

---

### Page Object Model

```javascript
// e2e/pages/LoginPage.js
export class LoginPage {
  constructor(page) {
    this.page = page;
    // Locators — prefer role/label selectors over CSS
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: /log in/i });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email, password) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async loginAndExpectRedirect(email, password, redirectPath = '/dashboard') {
    await this.login(email, password);
    await this.page.waitForURL(`**${redirectPath}`);
  }
}

// e2e/pages/DashboardPage.js
export class DashboardPage {
  constructor(page) {
    this.page = page;
    this.heading = page.getByRole('heading', { name: /dashboard/i });
    this.userMenu = page.getByRole('button', { name: /user menu/i });
    this.logoutButton = page.getByRole('menuitem', { name: /log out/i });
  }

  async expectLoaded() {
    await expect(this.heading).toBeVisible();
  }

  async logout() {
    await this.userMenu.click();
    await this.logoutButton.click();
    await this.page.waitForURL('**/login');
  }
}
```

---

### Authentication Fixture

```javascript
// e2e/fixtures/auth.js — avoid logging in via UI for every test
import { test as base } from '@playwright/test';
import { request } from '@playwright/test';

// Reuse authenticated state across tests
export const test = base.extend({
  // Authenticated page — session persisted in storageState
  authenticatedPage: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: 'e2e/.auth/user.json',
    });
    const page = await context.newPage();
    await use(page);
    await context.close();
  },
});

// e2e/setup/auth.setup.js — run once before tests
import { test as setup } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage.js';

setup('authenticate', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.loginAndExpectRedirect(
    process.env.TEST_USER_EMAIL,
    process.env.TEST_USER_PASSWORD
  );

  // Save browser state (cookies, localStorage) for reuse
  await page.context().storageState({ path: 'e2e/.auth/user.json' });
});
```

---

### Writing E2E Tests

```javascript
// e2e/auth.spec.js
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage.js';
import { DashboardPage } from './pages/DashboardPage.js';

test.describe('Authentication', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);
    const dashboardPage = new DashboardPage(page);

    await loginPage.goto();
    await loginPage.login(
      process.env.TEST_USER_EMAIL,
      process.env.TEST_USER_PASSWORD
    );

    await dashboardPage.expectLoaded();
  });

  test('shows error for invalid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page);

    await loginPage.goto();
    await loginPage.login('wrong@example.com', 'wrongpassword');

    await expect(loginPage.errorMessage).toContainText(/invalid email or password/i);
    await expect(page).toHaveURL(/\/login/);
  });

  test('user can log out', async ({ page }) => {
    // Use saved auth state — no UI login
    const context = await page.context().browser().newContext({
      storageState: 'e2e/.auth/user.json'
    });
    const authPage = await context.newPage();
    const dashboardPage = new DashboardPage(authPage);

    await authPage.goto('/dashboard');
    await dashboardPage.logout();

    await expect(authPage).toHaveURL(/\/login/);
    await context.close();
  });
});

// e2e/posts.spec.js — using auth fixture
import { test } from './fixtures/auth.js';
import { expect } from '@playwright/test';

test.describe('Post creation', () => {
  test('user can create and publish a post', async ({ authenticatedPage: page }) => {
    await page.goto('/posts/new');

    await page.getByLabel('Title').fill('My E2E Test Post');
    await page.getByLabel('Content').fill('This is the content of my post.');

    await page.getByRole('button', { name: /publish/i }).click();

    // Wait for navigation to post detail
    await page.waitForURL(/\/posts\/[\w-]+/);

    await expect(page.getByRole('heading', { name: 'My E2E Test Post' })).toBeVisible();
    await expect(page.getByText('This is the content of my post.')).toBeVisible();
    await expect(page.getByText(/published/i)).toBeVisible();
  });
});
```

---

### API-Assisted Setup

```javascript
// Speed up tests by using API to create data instead of UI
import { test } from './fixtures/auth.js';
import { expect } from '@playwright/test';

test('user can edit their post', async ({ authenticatedPage: page, request }) => {
  // Create test data via API — much faster than clicking through UI
  const res = await request.post('/api/v1/posts', {
    headers: { Authorization: `Bearer ${process.env.TEST_TOKEN}` },
    data: { title: 'Post To Edit', content: 'Original content' }
  });
  const { data: post } = await res.json();

  // Now test the UI behavior
  await page.goto(`/posts/${post.id}/edit`);
  await page.getByLabel('Title').fill('Updated Title');
  await page.getByRole('button', { name: /save/i }).click();

  await expect(page.getByRole('heading', { name: 'Updated Title' })).toBeVisible();
});
```

---

### CI Configuration

```yaml
# .github/workflows/e2e.yml
- name: Install Playwright browsers
  run: npx playwright install --with-deps chromium

- name: Run E2E tests
  run: npx playwright test
  env:
    BASE_URL: ${{ env.STAGING_URL }}
    TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
    TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

- name: Upload Playwright report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report
    path: playwright-report/
    retention-days: 7
```

---

### Selector Priority

```
1. getByRole()     — best: semantic, accessible, resilient to refactoring
2. getByLabel()    — good for form inputs
3. getByText()     — good for buttons and headings
4. getByTestId()   — acceptable when no semantic option exists (data-testid="x")
5. CSS selectors   — avoid: brittle, breaks on any DOM change
6. XPath           — avoid: unreadable, maintenance nightmare
```
