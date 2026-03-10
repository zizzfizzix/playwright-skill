---
name: playwright-scaffold
description: Set up Playwright in a project from scratch. Explores the project to detect framework, port, scripts, and required env vars, then installs @playwright/test, generates playwright.config.ts, and creates CI workflow files for GitHub Actions and GitLab CI. Use this before writing specs with playwright-spec.
argument-hint: [project-path]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm install*), Bash(npx playwright*)
---

# Playwright Scaffold

I set up Playwright infrastructure in a project: install `@playwright/test`, generate `playwright.config.ts`, and create CI workflow files. I do **not** write spec files — use `playwright-spec` for that.

**CRITICAL WORKFLOW - Follow these steps in order:**

## Step 1: Explore the Project

**Use your Read, Glob, and Grep tools — not shell commands — to explore the codebase.**

### Package & Config

```
Read: <project-root>/package.json          — framework, dev/build/start scripts, port, deps
Glob: <project-root>/playwright.config.*   — existing Playwright setup (skip if already present)
Read: <project-root>/.env.example          — what env vars the app requires (or .env.local.example)
Read: <project-root>/.env                  — actual values if present (never commit secrets)
```

Extract:
- **Framework**: Next.js, Vite, CRA, Express, Remix, etc.
- **Dev server command**: exact script name (`dev`, `start`, `serve`)
- **Build command**: exact script name (`build`) — needed for CI
- **Production serve command**: `start`, `preview`, etc. — what runs after `build` in CI
- **Port**: from scripts, `.env`, or framework defaults (Next.js=3000, Vite=5173, CRA=3000)
- **Required env vars**: everything in `.env.example` — the CI workflow must supply all of them
- **`@playwright/test` in devDependencies**: skip install if already present

If `playwright.config.*` already exists, skip Steps 2–3 and tell the user.

## Step 2: Install @playwright/test (if not present)

```bash
cd <project-root>
npm install --save-dev @playwright/test
npx playwright install chromium
```

## Step 3: Generate playwright.config.ts (if not present)

Write `playwright.config.ts` to the project root:

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

// Single source of truth for the dev server URL.
// Override with BASE_URL env var in CI to point at a deployed environment.
const BASE_URL = process.env.BASE_URL || 'http://localhost:3000'; // <-- adjust port to match your app

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['list'],
    ['html', { open: 'never' }],
  ],
  use: {
    baseURL: BASE_URL,
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'on-first-retry',
    headless: true,
  },
  projects: [
    // If the app has authentication, add a setup project (see Auth Patterns below).
    // Remove these two lines for apps with no login:
    { name: 'setup', testMatch: /.*\.setup\.ts/ },

    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'e2e/.auth/user.json', // remove if no auth
      },
      dependencies: ['setup'], // remove if no auth
    },
  ],
  webServer: {
    // In CI: use the production server (app must already be built — see CI workflow)
    // Locally: use the dev server for fast iteration
    // Adjust command for your framework:
    //   Next.js:  CI ? 'npm run start' : 'npm run dev'
    //   Vite:     CI ? 'npm run preview' : 'npm run dev'
    //   CRA:      CI ? 'serve -s build' : 'npm run start'
    command: process.env.CI ? 'npm run start' : 'npm run dev',
    url: BASE_URL,
    reuseExistingServer: !process.env.CI,
    stdout: 'ignore',
    stderr: 'pipe',
  },
});
```

**Key CI settings:**
- `forbidOnly`: Fails CI if `.only` is accidentally committed
- `retries: 2`: Handles flakiness in CI
- `workers: 1`: Avoids resource contention on CI runners
- `headless: true`: No display required in CI
- `screenshot: 'only-on-failure'`, `video/trace: 'on-first-retry'`: capture artifacts on failure
- `BASE_URL` constant: port defined once — both `use.baseURL` and `webServer.url` reference it
- `webServer.command`: serves the **production build** in CI

## Step 4: Add .gitignore Entries

```
# Playwright test output
/test-results/
/playwright-report/
# Saved auth state — contains session tokens, never commit
/e2e/.auth/
```

## Step 5: Generate CI Workflows

### GitHub Actions

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci

      - uses: actions/cache@v4  # cache browsers — avoids re-downloading ~200MB
        id: playwright-cache
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ hashFiles('package-lock.json') }}

      - name: Install Playwright browsers
        if: steps.playwright-cache.outputs.cache-hit != 'true'
        run: npx playwright install --with-deps chromium

      - name: Build app  # build first — fails clearly if TypeScript/build errors exist
        run: npm run build
        # env: NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}  # add build-time vars

      - name: Run E2E tests
        run: npx playwright test
        env:
          CI: true
          TEST_EMAIL: ${{ secrets.TEST_EMAIL }}
          TEST_PASSWORD: ${{ secrets.TEST_PASSWORD }}
          # DATABASE_URL: ${{ secrets.DATABASE_URL }}
          # NEXTAUTH_SECRET: ${{ secrets.NEXTAUTH_SECRET }}

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

### GitLab CI

```yaml
# .gitlab-ci.yml (e2e stage)
e2e-tests:
  image: mcr.microsoft.com/playwright:v1.58.2-jammy
  stage: test
  script:
    - npm ci
    - npm run build
    - npx playwright test
  variables:
    CI: "true"
    TEST_EMAIL: $TEST_EMAIL
    TEST_PASSWORD: $TEST_PASSWORD
    # DATABASE_URL: $DATABASE_URL
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 7 days
```

### CI Environment Variables

Read `.env.example` in Step 1 to identify everything required, then ensure all of it is available in CI.

**Always needed:**
| Variable | Source | Notes |
|---|---|---|
| `TEST_EMAIL` | CI secret | Login credentials for test user |
| `TEST_PASSWORD` | CI secret | Login credentials for test user |

**Usually needed (check `.env.example`):**
| Variable | Source | Notes |
|---|---|---|
| `DATABASE_URL` | CI secret | If the app connects to a real DB in tests |
| `NEXTAUTH_SECRET` / `JWT_SECRET` | CI secret | Session signing key — app won't start without it |
| `NEXT_PUBLIC_API_URL` | CI var or secret | If the frontend calls an external API |
| App-specific secrets | CI secret | Anything else in `.env.example` |

**Never hardcode secrets in the workflow file.** Set them in:
- GitHub Actions: Settings → Secrets and variables → Actions → New repository secret
- GitLab: Settings → CI/CD → Variables

**After scaffolding:** Tell the user exactly which variables they need to add as CI secrets before the workflow will work.

---

## Tips

- **Check first** — if `playwright.config.*` already exists, skip config generation and tell the user
- **Build before testing in CI** — `npm run build` as a separate CI step; `webServer.command` in CI should serve the built app, not the dev server
- **Use `baseURL`** — write `page.goto('/')` not `page.goto('http://localhost:3000/')` so tests work in any environment
- **`forbidOnly`** — catches accidental `.only` in CI before it skips the entire suite
- **`webServer` config** — use `reuseExistingServer: !process.env.CI` so CI always starts fresh but local dev reuses running servers
- **Environment variables** — read `.env.example` and tell the user which variables must be set as CI secrets
