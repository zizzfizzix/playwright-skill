---
name: playwright-spec
description: Write CI-ready Playwright spec files for an existing project. Explores routes, auth flow, and key components to write persistent *.spec.ts files that match the real app. Requires playwright.config.ts to already exist — run playwright-scaffold first if it doesn't.
argument-hint: [project-path or spec-file] [--headed] [--grep <pattern>]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm install*), Bash(npx playwright*)
---

# Playwright Spec Writer

I write complete, CI-ready `*.spec.ts` files using `@playwright/test`. Tests are written **to the project's test directory** so they persist, are committed to source control, and run reliably in CI pipelines.

**Prerequisite:** `playwright.config.ts` must already exist. Run `playwright-scaffold` first if it doesn't.

**CRITICAL WORKFLOW - Follow these steps in order:**

## Step 1: Explore the Project

**Use your Read, Glob, and Grep tools — not shell commands — to explore the codebase.**

Tests are only production-ready if they match the actual app. Generic templates will fail. Read the code first.

### Package & Config

```
Read: <project-root>/package.json          — framework, dev/build/start scripts, port, deps
Read: <project-root>/playwright.config.ts  — testDir, baseURL, auth setup
Read: <project-root>/.env.example          — env vars needed for test credentials
```

Extract:
- **testDir** from `playwright.config.ts` — where to write spec files
- **Port / baseURL** — so you know what the app listens on locally
- **Auth setup** — does the config include a `setup` project with `storageState`?

### Routes & Pages

Read the routing layer to know what URLs exist — don't guess:

```
# Next.js App Router
Glob: <project-root>/app/**/page.tsx

# Next.js Pages Router
Glob: <project-root>/pages/**/*.tsx

# React Router (find the router definition)
Grep: "createBrowserRouter|<Route|<Routes" in src/

# Express / Fastify
Grep: "app\.get|router\.get|fastify\.get" in src/
```

### Auth Flow

Read the actual login UI, not docs — you need the real field labels and post-login URL:

```
Grep: "login|signin|sign-in" in src/ for the login page/component
Read: the login component/page file
```

Look for:
- Actual label text on email/username and password fields
- The submit button's text
- Where the app redirects after successful login (e.g., `/dashboard`, `/home`, `/app`)
- Whether auth uses a form, OAuth redirect, magic link, etc.

### Key User Flows

Read the main components to understand what the app actually does:

```
Glob: <project-root>/src/components/**/*.tsx  (or .jsx, .vue, .svelte)
Read: the nav/layout component — reveals all main sections
Read: key feature components for the flows you'll test
```

### Existing Tests

Avoid duplicating coverage that already exists:

```
Glob: <project-root>/e2e/**/*.spec.ts
Glob: <project-root>/cypress/**/*.cy.ts
Grep: "test\(|it\(|describe\(" in src/ — unit/integration test coverage
```

After this exploration you should know:
- Every URL you'll navigate to
- The actual text of labels, buttons, and headings you'll interact with
- What the app does after login
- What's already tested vs. what's missing

## Step 2: Plan the Test Suite

Based on the project, decide:
- **Where to write tests**: Use `testDir` from `playwright.config.ts`
- **What to test**: Key user flows (auth, navigation, critical features) inferred from the codebase
- **Auth pattern**: Use `auth.setup.ts` + `storageState` if the config already includes a setup project

## Step 3: Write Test Spec Files

Write spec files to `<testDir>/` using `@playwright/test` syntax:

```typescript
// e2e/homepage.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Homepage', () => {
  test('loads with correct title', async ({ page }) => {
    await page.goto('/');
    await expect(page).toHaveTitle(/My App/);
  });

  test('renders main navigation', async ({ page }) => {
    await page.goto('/');
    await expect(page.getByRole('navigation')).toBeVisible();
  });

  test('hero CTA is clickable', async ({ page }) => {
    await page.goto('/');
    await page.getByRole('link', { name: /get started/i }).click();
    await expect(page).not.toHaveURL('/');
  });
});
```

### Authentication Flow

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: /sign in/i }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByTestId('user-menu')).toBeVisible();
  });

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('wrong@example.com');
    await page.getByLabel('Password').fill('wrongpassword');
    await page.getByRole('button', { name: /sign in/i }).click();

    await expect(page.getByRole('alert')).toContainText(/invalid/i);
  });
});
```

### Auth Setup Project (if playwright.config.ts has a setup project)

```typescript
// e2e/auth.setup.ts
import { test as setup } from '@playwright/test';

const authFile = 'e2e/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);     // ACTUAL label from component
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: /sign in/i }).click();
  await page.waitForURL('**/dashboard');                             // ACTUAL post-login URL
  await page.context().storageState({ path: authFile });
});
```

To opt a test **out** of auth (e.g., testing the login page itself):
```typescript
test.use({ storageState: { cookies: [], origins: [] } });
```

## Step 4: Run Tests to Validate

```bash
# Run all tests
cd <project-root> && npx playwright test

# Run a specific spec file
cd <project-root> && npx playwright test e2e/homepage.spec.ts

# Run with headed browser for local debugging
cd <project-root> && npx playwright test --headed
```

**If tests fail:**
- Run with `--headed` to watch the browser and see what's actually on screen
- Read the failure output — Playwright reports the expected vs. actual value and a screenshot path
- The most common causes:
  - **Wrong selector**: re-read the component source to find the actual label/role/text
  - **Wrong URL**: check the actual redirect after login by reading auth code
  - **App not started**: ensure `webServer.command` in config matches the actual dev script
  - **Timing**: if an element isn't found, add `await expect(locator).toBeVisible()` before interacting
- Fix the test to match what the app actually does — never change assertions to make failures pass artificially

---

## Selector Best Practices

Prefer in this order (most to least resilient):

```typescript
// 1. Role + name (best: semantic, accessible)
page.getByRole('button', { name: 'Submit' })
page.getByRole('link', { name: 'About' })
page.getByRole('textbox', { name: 'Email' })

// 2. Label (form inputs)
page.getByLabel('Email address')

// 3. Test ID (explicit, stable)
page.getByTestId('submit-btn')

// 4. Text content
page.getByText('Welcome back')

// 5. Placeholder
page.getByPlaceholder('Search...')

// Avoid: CSS selectors, XPath, positional selectors
// Only use as last resort:
page.locator('.specific-class')
page.locator('#specific-id')
```

---

## Tips

- **Always explore first** — read `playwright.config.ts`, route files, auth components before writing anything
- **Use `baseURL`** — write `page.goto('/')` not `page.goto('http://localhost:3000/')` so tests work in any environment
- **Prefer role selectors** — `getByRole`, `getByLabel`, `getByTestId` over CSS selectors; read the actual component to find real labels
- **`forbidOnly`** — catches accidental `.only` in CI before it skips the entire suite
- **Test isolation** — each test should be independent; use `beforeEach` for shared setup, not shared state
- **`storageState` for auth** — use the setup project pattern to log in once and reuse auth state across all tests; per-test login multiplies CI time
- **No hardcoded waits** — never use `page.waitForTimeout()`; use `waitForURL`, locator assertions, or `expect.poll()` for async conditions

For advanced API reference (network mocking, POM, visual testing, accessibility), see [API_REFERENCE.md](API_REFERENCE.md).
