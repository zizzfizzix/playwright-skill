---
name: playwright-debug
description: Diagnose and fix failing Playwright tests. Reads test output, error messages, and failure artifacts (screenshots, traces), traces the root cause in the app source, and fixes the specs to match actual application behaviour. Use when tests are failing and you need to understand why and fix them.
argument-hint: [project-path] [--test-output <file>] [--grep <pattern>]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npx playwright*)
---

# Playwright Debug

I diagnose and fix failing Playwright tests. I read test output and failure artifacts, trace the root cause back to the app source or the test itself, and fix the spec — never by weakening assertions, always by making the test match reality.

**CRITICAL PRINCIPLE:** Fix the test to match what the app actually does. Never change an assertion to pass artificially (e.g., removing an `expect`, changing a URL match to `*`). If the app is broken, say so and ask whether to fix the app or skip the test.

**CRITICAL WORKFLOW - Follow these steps in order:**

## Step 1: Get the Failure Output

If the user hasn't pasted it, run the failing test(s) to capture fresh output:

```bash
cd <project-root>

# Run all failing tests
npx playwright test

# Run a specific file or pattern
npx playwright test e2e/auth.spec.ts
npx playwright test --grep "login"

# Run headed to watch the browser
npx playwright test --headed e2e/auth.spec.ts
```

Capture:
- The full error message and stack trace
- The expected vs. actual value
- The screenshot path (if `screenshot: 'only-on-failure'` is configured)
- The test name and file

## Step 2: Read the Failure Artifacts

```
Read: <project-root>/test-results/**/test-failed-*.png   — screenshot at point of failure
Glob: <project-root>/test-results/**/*.zip               — trace file (open with npx playwright show-trace)
```

If a trace file exists, run:
```bash
npx playwright show-trace <path-to-trace.zip>
```

From the screenshot and trace, identify:
- What was actually on screen vs. what the test expected
- Whether the element existed but had different text/role/label
- Whether the page navigated to the wrong URL
- Whether the app threw an error or showed an unexpected state

## Step 3: Read the Relevant Source

Based on the failure, read the actual app source to find the root cause:

```
# Wrong selector — read the component
Read: <source-file-for-the-component>

# Wrong URL after login — read the auth handler
Grep: "redirect|push|replace|router" in the auth handler file

# App not starting — read package.json scripts
Read: <project-root>/package.json

# Missing env var — read .env.example
Read: <project-root>/.env.example

# Wrong page content — read the route/page component
Read: <route-or-page-file>
```

Read the `playwright.config.ts` too if the failure looks config-related:
```
Read: <project-root>/playwright.config.ts
```

## Step 4: Identify the Root Cause

Common causes:

| Symptom | Root cause | Fix |
|---|---|---|
| `locator not found` | Selector doesn't match DOM | Read component, use correct label/role/text |
| `expect(page).toHaveURL` mismatch | Wrong post-action URL | Read route handlers / auth redirect |
| `Timeout waiting for element` | Element never appears (wrong state/timing) | Add `waitForURL` or `expect(locator).toBeVisible()` before interacting |
| `net::ERR_CONNECTION_REFUSED` | App server not running | Check `webServer.command` in config vs. actual dev script |
| `storageState file not found` | Auth setup didn't run or failed | Check `auth.setup.ts` — is it matching the actual login flow? |
| `forbidOnly` failure | `.only` left in a test | Remove all `.only` calls |
| Test passes locally, fails in CI | Missing env var / different build | Check CI secrets and `webServer.command` |
| Flaky test | Race condition / missing await | Add explicit `waitFor` or assertion before the flaky action |

## Step 5: Fix the Test

Edit the spec file(s) to match the actual application behaviour:

```typescript
// Before (wrong selector — guessed label)
await page.getByLabel('Email Address').fill('user@example.com');

// After (correct — read from component source)
await page.getByLabel('Work email').fill('user@example.com');
```

```typescript
// Before (wrong post-login URL)
await expect(page).toHaveURL('/dashboard');

// After (correct — read from auth redirect)
await expect(page).toHaveURL('/app/home');
```

```typescript
// Before (no wait — element may not be visible yet)
await page.getByRole('button', { name: 'Submit' }).click();

// After (wait for element first)
await expect(page.getByRole('button', { name: 'Submit' })).toBeVisible();
await page.getByRole('button', { name: 'Submit' }).click();
```

## Step 6: Re-run to Confirm

```bash
cd <project-root> && npx playwright test <fixed-spec-file>
```

If still failing, go back to Step 2. Do not mark as done until the test passes.

---

## Debugging Tools

### Run with Inspector

```bash
npx playwright test --debug e2e/auth.spec.ts
```

Opens the Playwright Inspector — step through the test, inspect the DOM, and try selectors interactively.

### Headed Mode + Slow Motion

```bash
npx playwright test --headed --slowmo=500 e2e/auth.spec.ts
```

Slows execution so you can see what's happening in the browser.

### Pause in Code

Add `await page.pause()` at the point of failure to freeze execution and open the inspector:

```typescript
await page.goto('/login');
await page.pause(); // stops here — inspect the DOM
await page.getByLabel('Email').fill('user@example.com');
```

Remove `pause()` before committing.

### Console & Error Monitoring

Add to the test or `beforeEach` to catch JS errors:

```typescript
page.on('console', msg => {
  if (msg.type() === 'error') console.log('Browser error:', msg.text());
});
page.on('pageerror', error => console.log('Uncaught error:', error));
```

### Generate a Codegen Trace

If you're unsure what selectors to use, record interactions against the running app:

```bash
cd <project-root> && npx playwright codegen http://localhost:3000
```

---

## Tips

- **Read before fixing** — always look at the component source and the actual DOM (via screenshot/inspector) before editing the test
- **One fix at a time** — change one thing, re-run, confirm it helps before changing more
- **Never weaken assertions** — if `toHaveURL('/dashboard')` fails, fix the URL to match reality; don't change it to `toHaveURL(/.*/)`
- **Flaky tests** — add explicit `expect(locator).toBeVisible()` or `waitForURL` rather than `waitForTimeout`; flakiness is always a missing wait
- **CI-only failures** — check env vars first (`TEST_EMAIL`, `TEST_PASSWORD`, secrets), then `webServer.command` (CI should serve the built app, not the dev server)
- **Auth failures** — re-read `auth.setup.ts` against the actual login component; label text and post-login URL must match exactly
