# playwright-skill

**A suite of Claude Code skills for end-to-end testing with Playwright**

Three focused [Claude Code Plugins](https://docs.claude.com/en/docs/claude-code/plugins) that cover the full Playwright workflow — scaffolding, spec writing, and debugging — each installable independently.

## Plugins

### [`playwright-scaffold`](plugins/playwright-scaffold/) · v1.0.0

Explores your project (framework, port, scripts, env vars), installs `@playwright/test`, generates `playwright.config.ts`, and creates CI workflow files for GitHub Actions and GitLab CI. Run this first, before writing specs.

### [`playwright-spec`](plugins/playwright-spec/) · v1.0.0

Writes CI-ready `*.spec.ts` files. Reads your routes, auth flow, and components to generate persistent spec files that match the real app. Requires `playwright.config.ts` — run `playwright-scaffold` first if it doesn't exist.

### [`playwright-debug`](plugins/playwright-debug/) · v1.0.0

Diagnoses and fixes failing Playwright tests. Reads test output and failure artifacts (screenshots, traces), traces the root cause in the app source, and fixes specs to match actual application behaviour — never by weakening assertions.

---

## Installation

```bash
# Add this repository as a Claude Code marketplace
/plugin marketplace add zizzfizzix/playwright-skill

# Install the plugins you need
/plugin install playwright-scaffold
/plugin install playwright-spec
/plugin install playwright-debug
```

---

## Typical Workflow

1. **Scaffold** — set up Playwright in a new project:
   ```
   /playwright-scaffold
   ```
   Claude installs `@playwright/test`, generates `playwright.config.ts`, and creates CI workflow files.

2. **Write specs** — generate test files for your app:
   ```
   /playwright-spec
   ```
   Claude explores your routes, auth, and components, then writes `*.spec.ts` files to your project's `e2e/` directory.

3. **Debug** — fix failing tests:
   ```
   /playwright-debug
   ```
   Claude reads the failure output and artifacts, traces the root cause, and fixes the specs.

---

## What Gets Written to Your Project

| File | Plugin | Purpose |
|------|--------|---------|
| `playwright.config.ts` | scaffold | CI-ready config with `forbidOnly`, retries, artifact capture |
| `.github/workflows/e2e.yml` | scaffold | GitHub Actions CI workflow |
| `.gitlab-ci.yml` (e2e job) | scaffold | GitLab CI job |
| `e2e/*.spec.ts` | spec | Test spec files using `@playwright/test` |
| `e2e/auth.setup.ts` | spec | Auth setup: logs in once, saves `storageState` |
| `.gitignore` entries | spec | Excludes `test-results/`, `playwright-report/`, `e2e/.auth/` |

Tests are committed to your project's source control and run via `npx playwright test`.

---

## Requirements

- Node.js ≥ 18
- `@playwright/test` (installed by `playwright-scaffold`)
- Chromium: `npx playwright install chromium` (run once after install)

---

## Running Tests

```bash
npx playwright test                         # run all tests
npx playwright test e2e/auth.spec.ts        # run a specific file
npx playwright test --headed                # watch the browser
npx playwright test --grep "login"          # filter by name
```

---

## Repository Structure

```
plugins/
├── playwright-scaffold/
│   ├── .claude-plugin/plugin.json
│   └── skills/playwright-scaffold/SKILL.md
├── playwright-spec/
│   ├── .claude-plugin/plugin.json
│   └── skills/playwright-spec/SKILL.md
└── playwright-debug/
    ├── .claude-plugin/plugin.json
    └── skills/playwright-debug/SKILL.md
.claude-plugin/marketplace.json
```

---

## Learn More

- [Claude Code Skills Documentation](https://docs.claude.com/en/docs/claude-code/skills)
- [Claude Code Plugins Documentation](https://docs.claude.com/en/docs/claude-code/plugins)
- [Playwright Documentation](https://playwright.dev/docs/intro)

## Contributing

Contributions are welcome. Fork the repository, create a feature branch, and submit a pull request. See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

Uses [Release Please](https://github.com/googleapis/release-please) with conventional commits scoped to the plugin name (e.g. `feat(playwright-spec): ...`).

## License

MIT — see [LICENSE](LICENSE) for details.
