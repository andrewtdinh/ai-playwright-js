# AI Playwright Test Lab

An end-to-end AI-powered test automation pipeline that turns plain-English user stories into running Playwright tests, analyzes failures with Claude, and produces a styled HTML report.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Project Structure](#project-structure)
3. [Installation](#installation)
4. [Workflow: Run All Tests](#workflow-run-all-tests)
5. [Workflow: Add a New Story and Test It](#workflow-add-a-new-story-and-test-it)
6. [Individual Commands Reference](#individual-commands-reference)
7. [UI Dashboard](#ui-dashboard)
8. [Claude Code Skill](#claude-code-skill)
9. [How AI Analysis Works](#how-ai-analysis-works)
10. [Roadmap](#roadmap)

---

## Tech Stack

- **Node.js + ES Modules**
- **Playwright** (`@playwright/test`)
- **Claude Code CLI** — no API keys needed, uses your Claude Code subscription
- **Vanilla Node `http` server** for the local UI (no framework)
- **Dark-mode HTML dashboard** with external CSS

---

## Project Structure

```
ai-playwright-js/
│
├── stories/                       # Plain-English test specifications (.md)
├── tests/                         # Auto-generated Playwright specs
│   └── .ai-backups/               # Backups created before AI fixes
│
├── analyze/
│   └── analyzeFailures.js         # AI failure analysis engine
├── utils/
│   ├── claudeClient.js            # Claude CLI subprocess helpers
│   └── testHelpers.js             # Shared test utilities
├── server/
│   └── server.mjs                 # Vanilla Node HTTP server
├── ui/                            # Frontend (index.html, ui.js, ui.css, modules/)
│
├── generateTest.js                # Converts stories → Playwright specs
├── playwright.config.mjs          # Config + JSON + HTML reporters
└── package.json                   # npm scripts for full automation
```

**Data flow:**
```
stories/*.md
  → generateTest.js
      → tests/*.spec.js
          → playwright test  →  playwright-report.json  +  playwright-report/
              → analyzeFailures.js
                  → ai-analysis.json  +  ai-report.html
```

---

## Installation

```bash
git clone https://github.com/andrewtdinh/ai-playwright-js.git
cd ai-playwright-js
npm install
npx playwright install
```

No API keys required. The project uses the **Claude Code CLI** (`claude` command) bundled with your Claude Code subscription. [Get Claude Code here](https://claude.ai/code) if you haven't installed it.

---

## Workflow: Run All Tests

Use this when you want to run the entire suite end-to-end.

**Step 1 — Run the full pipeline**

```bash
npm run ai:flow
```

This does four things in sequence:
1. Generates Playwright specs from every story in `stories/`
2. Runs the full Playwright suite
3. Analyzes any failures with AI
4. Builds and opens `ai-report.html` in your browser

**Step 2 — Review the results**

- `ai-report.html` — AI dashboard: plain-English explanations, root causes, fix suggestions, flakiness tips
- `playwright-report/` — Traditional Playwright HTML report with screenshots
- `playwright-report.json` — Raw machine-readable output

---

## Workflow: Add a New Story and Test It

Use this when you've written a new story and only want to generate and run that one test.

**Step 1 — Write your story**

Create `stories/<slug>.md`. The filename stem becomes the spec filename.

```markdown
Title: Login - valid credentials

Base URL: https://the-internet.herokuapp.com

As a user, I want to log in with valid credentials so I can access the secure area.

Acceptance criteria:
- Navigate to `/login`
- Fill in username `tomsmith`
- Fill in password `SuperSecretPassword!`
- Click the Login button
- Expect redirect to `/secure`
- Expect the success flash message to contain: `You logged into a secure area!`
```

Requirements:
- Always include `Base URL:` — the generator uses it to build full URLs
- Use relative paths in acceptance criteria (`/login`, not the full URL)
- The slug (filename stem) maps directly: `login.md` → `tests/login.spec.js`

**Step 2 — Generate the test**

```bash
npm run ai:gen -- --story login.md
```

This creates `tests/login.spec.js`. Review it and adjust any locators if needed.

**Step 3 — Run only that test**

```bash
npx playwright test tests/login.spec.js
```

**Step 4 — If it fails, get AI analysis**

```bash
npx playwright test tests/login.spec.js
npm run ai:report
```

Or run the full pipeline for steps 2–4 in one command:

```bash
npm run ai:flow
```

---

## Individual Commands Reference

```bash
# Generate tests
npm run ai:gen                            # all stories
npm run ai:gen -- --story login.md        # one story

# Run tests
npm run ai:test                           # all tests
npx playwright test tests/login.spec.js   # one test

# Analyze failures (requires playwright-report.json)
npm run ai:analyze                        # JSON output only
npm run ai:report                         # JSON + HTML report (auto-opens)

# Full pipeline
npm run ai:flow                           # all stories + all tests + analyze + report

# UI server
npm run ui                                # http://localhost:5173

# Unit tests (Node built-in runner, not Playwright)
npm test
npm run test:watch
```

---

## UI Dashboard

```bash
npm run ui
```

Open `http://localhost:5173`.

**Run Bar** — trigger the full pipeline or individual steps (Generate / Run Tests / Analyze) with a live status chip.

**Editor** — write or edit stories and tests with validation, upload up to 5 stories at once, or use the AI Story Assistant wizard to build a story from requirements and expected outcomes.

**Saved Stories / Generated Tests** — browse, edit, delete, or run individual stories or tests from the sidebar.

**Run Console** — step-by-step progress (Validate → Save → Generate → Run → Analyze → Report), live logs, and a **Fix with AI** button on failed tests that auto-applies a Claude-generated fix, creates a backup in `tests/.ai-backups/`, re-runs the test, and shows a fix summary.

**Report Tabs** — embedded AI analysis report and traditional Playwright HTML report with screenshots, side by side.

---

## Claude Code Skill

This repo ships a Claude Code skill at `.claude/skills/playwright/SKILL.md`.

When working inside Claude Code, type:

```
/playwright
```

This loads the skill and gives Claude full context about this project: the story format, spec file conventions, all available commands, locator best practices, wait strategies, POM guidance, and an ad-hoc exploration pattern for quick locator debugging.

**When to use it:**
- Writing a new story or spec file
- Debugging a failing test
- Asking Claude to explain a locator or wait strategy
- Getting help with the full pipeline

---

## How AI Analysis Works

After Playwright executes the tests, failure metadata is passed to Claude via the CLI:

- Error message and full stack trace
- Test title and file path
- stdout/stderr logs
- The original user story (for context about intent)

Claude returns a structured HTML snippet for each failure containing:

1. **Plain-English Explanation** — why it likely failed, grounded in the real behavior of the page under test
2. **Probable Root Causes** — 2–3 concrete technical causes
3. **Suggested Test Fixes** — specific Playwright code improvements
4. **Flakiness Mitigation** — ways to reduce intermittent failures

Output is written to `ai-analysis.json` (structured data) and `ai-report.html` (styled dashboard).

---

## Roadmap

- Multi-story batch generation
- Test-to-story reverse engineering
- Flaky test scoring over time
- Hit-map UI of frequent failures
- CI pipeline integration
- Slack/Teams bot that posts AI insights
