# playwright Skill

You are a Playwright (@playwright/test) expert for this project. Use this guide whenever the user asks you to write, run, fix, or explore Playwright tests.

## Project Data Flow

```
stories/*.md
  → node generateTest.js [--story <file>]
      → tests/*.spec.js
          → npx playwright test [tests/<name>.spec.js]
              → node analyze/analyzeFailures.js [--html]
                  → ai-analysis.json  +  ai-report.html
```

Full pipeline shortcut:
```bash
npm run ai:flow
```

## Story Format

Story files live in `stories/<slug>.md`. The slug becomes the spec filename.

```markdown
Title: <Descriptive title>

Base URL: https://example.com

As a user, I want to <goal>.

Acceptance criteria:
- Navigate to `/path`
- Fill in <field> `value`
- Click the <button>
- Expect <assertion>
```

Always include `Base URL:`. Use relative paths (`/login`) in acceptance criteria — the generator prepends the base URL.

## Test File Format

Tests live in `tests/<slug>.spec.js`. One `test()` block per file, named after the slug.

```js
import { test, expect } from '@playwright/test';

test('<slug>', async ({ page }) => {
  await page.goto('https://example.com/path');
  await page.getByLabel('Username').fill('tomsmith');
  await page.getByRole('button', { name: 'Login' }).click();
  await expect(page).toHaveURL('https://example.com/secure');
  await expect(page.getByRole('alert')).toContainText('You logged into a secure area!');
});
```

Rules:
- Always import from `'@playwright/test'`
- Use the `async ({ page }) => {}` fixture — never instantiate `chromium.launch()` in spec files
- Prepend the Base URL to relative paths explicitly
- One `test(...)` block per file (unless using POM — see below)
- All Playwright calls must be `await`ed

## Commands Reference

```bash
# Generate test(s) from story file(s)
npm run ai:gen                            # all stories
npm run ai:gen -- --story login.md        # one story

# Run tests
npm run ai:test                           # all tests
npx playwright test tests/login.spec.js   # one test

# Analyze failures (requires playwright-report.json)
npm run ai:analyze                        # JSON output only
npm run ai:report                         # JSON + HTML report (auto-opens browser)

# Full pipeline: generate → test → analyze → open report
npm run ai:flow

# UI server (http://localhost:5173)
npm run ui

# Unit tests (Node built-in runner, not Playwright)
npm test
```

All commands run from the project root.

## Ad-hoc Exploration

When you need to quickly test a locator or explore a page without adding a permanent test, write a throwaway script to `/tmp/` and run it directly — do NOT place it in `tests/`:

```js
// /tmp/pw_scratch.mjs
import { chromium } from 'playwright';

const browser = await chromium.launch({ headless: false, slowMo: 100 });
const page = await browser.newPage();
await page.goto('https://the-internet.herokuapp.com/login');
console.log(await page.title());
// inspect, screenshot, etc.
await browser.close();
```

```bash
node /tmp/pw_scratch.mjs
```

Clean up after: `rm /tmp/pw_scratch.mjs`

Use `headless: false` and `slowMo: 100` when exploring so you can see what's happening.

Note: `playwright` (not `@playwright/test`) is the low-level package used for scratch scripts. The `@playwright/test` runner is used for actual spec files in `tests/`.

## Page Object Model (POM)

This project does not currently use POM. Introduce it only when a page is shared across multiple spec files. Structure:

```
pages/
  LoginPage.js       # LoginPage class
tests/
  login.spec.js      # imports LoginPage
  secure.spec.js     # imports LoginPage
```

Page object pattern:
```js
// pages/LoginPage.js
export class LoginPage {
  constructor(page) {
    this.page = page;
    this.username = page.getByLabel('Username');
    this.password = page.getByLabel('Password');
    this.submit = page.getByRole('button', { name: 'Login' });
  }

  async login(username, password) {
    await this.page.goto('https://the-internet.herokuapp.com/login');
    await this.username.fill(username);
    await this.password.fill(password);
    await this.submit.click();
  }
}
```

Keep assertions in spec files, not page objects.

## Locator Strategy

Prefer in this order:
1. `getByRole('button', { name: 'Submit' })` — semantic, accessible
2. `getByLabel('Username')` — for inputs with labels
3. `getByText('Welcome')` — for visible text
4. `getByPlaceholder('Enter email')` — for inputs by placeholder
5. `locator('#id')` or `locator('[data-testid=foo]')` — only if semantic locators aren't available
6. Never use fragile CSS like `nth-child`, positional XPath, or long descendant selectors

## Wait Strategies

Never use `page.waitForTimeout()` (hardcoded sleep). Use:

```js
await page.waitForURL('**/secure');            // after navigation
await page.waitForLoadState('networkidle');    // after heavy page loads
await expect(locator).toBeVisible();           // auto-waits up to default timeout
await expect(locator).toContainText('...');    // auto-waits
```

Playwright's `expect()` assertions auto-retry for up to 5 seconds by default — lean on them.

## File Naming Convention

| Story file | Test file | Test name |
|---|---|---|
| `stories/login.md` | `tests/login.spec.js` | `'login'` |
| `stories/add_remove_elements.md` | `tests/add_remove_elements.spec.js` | `'add_remove_elements'` |

The slug is always the story filename stem (no extension).

## Workflow for New Tests

1. Write `stories/<slug>.md` with Title, Base URL, and acceptance criteria
2. Run `npm run ai:gen -- --story <slug>.md` to generate the test
3. Review `tests/<slug>.spec.js` — edit locators or assertions if needed
4. Run `npx playwright test tests/<slug>.spec.js` to verify
5. If it fails, run `npm run ai:report` for AI diagnosis
