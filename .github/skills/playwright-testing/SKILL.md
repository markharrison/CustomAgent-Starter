# Playwright Testing Skill

Conventions and patterns for running Playwright MCP functional tests in agent pipelines.
Read this skill before writing any Playwright test steps.

---

## Persisting Tests as a Script — REQUIRED

Playwright MCP runs tests interactively during the agent session. **When the session ends,
those tests are gone.** To make functional tests replayable outside the agent, the tester
MUST write a `tests/playwright.spec.js` file as a deliverable.

This file should mirror every check performed via the MCP tools:

```js
// @ts-check
const { test, expect } = require('@playwright/test');

// Base URL assumes live-server or equivalent running on port 8080
const BASE_URL = 'http://localhost:8080';

test.describe('Page functional tests', () => {
  test('page loads without console errors', async ({ page }) => {
    const errors = [];
    page.on('console', msg => { if (msg.type() === 'error') errors.push(msg.text()); });
    await page.goto(BASE_URL);
    expect(errors).toHaveLength(0);
  });

  test('expected content is visible', async ({ page }) => {
    await page.goto(BASE_URL);
    // Add content-specific assertions here
  });

  test('interactive elements respond to clicks', async ({ page }) => {
    await page.goto(BASE_URL);
    // Add interaction assertions here
  });
});
```

Save to `./tests/playwright.test.js` — **committed to the repo**, not gitignored.
Screenshots taken during the MCP session go to `./test-artifacts/` (gitignored).

---

## Protocol Rule — CRITICAL

**Playwright blocks `file://` URLs.** Navigating directly to a local file path will fail
with an "Access to 'file:' URL is blocked" error.

Always serve pages over HTTP before navigating. The only allowed protocols are
`http:`, `https:`, `about:`, and `data:`.

---

## Web Server: live-server

Use `live-server` as the local HTTP server. It requires no installation beyond `npx`.

### Start command

```bash
npx live-server <dir> --port=8080 --no-browser --quiet
```

| Argument       | Purpose                                          |
| -------------- | ------------------------------------------------ |
| `<dir>`        | Directory to serve (e.g. `src`)                  |
| `--port=8080`  | Fixed port — always use 8080 for consistency     |
| `--no-browser` | Prevent live-server from opening a browser tab   |
| `--quiet`      | Suppress live-server stdout noise                |

### Startup sequence

1. Run the command with `execute/runInTerminal` — **background: true**
2. Wait 2 seconds using `mcp_playwright_browser_wait_for` (`time: 2`) for the server to be ready
3. Navigate to `http://localhost:8080/` using `mcp_playwright_browser_navigate`

### Teardown

Kill the live-server background process after all Playwright tests complete.
Do not leave it running — it will hold port 8080 for the rest of the session.

### Example port conflict

If port 8080 is already in use, increment to 8081 and update the navigate URL to match.

---

## Playwright MCP Tool Reference

| Step                        | Tool                                     | Notes                                      |
| --------------------------- | ---------------------------------------- | ------------------------------------------ |
| Wait for server ready       | `mcp_playwright_browser_wait_for`        | `time: 2` (seconds)                        |
| Navigate to page            | `mcp_playwright_browser_navigate`        | Always `http://localhost:8080/`            |
| Check console messages      | `mcp_playwright_browser_console_messages`| Use `level: "error"` to catch JS errors    |
| Accessibility snapshot      | `mcp_playwright_browser_snapshot`        | Use for content/layout verification        |
| Take screenshot             | `mcp_playwright_browser_take_screenshot` | Use `filename` param — see Artifacts below |
| Click element               | `mcp_playwright_browser_click`           | Use `ref` from snapshot for precision      |
| Evaluate JS                 | `mcp_playwright_browser_evaluate`        | Check DOM state, computed styles           |
| Hover element               | `mcp_playwright_browser_hover`           | Test hover interactions                    |
| Fill form field             | `mcp_playwright_browser_fill_form`       | For input/form testing                     |
| Resize viewport             | `mcp_playwright_browser_resize`          | Test responsive layouts                    |

### Snapshot vs Screenshot

| Tool        | Use when                                                    |
| ----------- | ----------------------------------------------------------- |
| `snapshot`  | Verifying content, element presence, accessibility — fast   |
| `screenshot`| Capturing visual state for reports or failure evidence      |

Prefer `snapshot` for assertions; use `screenshot` to record before/after states.

---

## Standard Test Sequence

Apply this sequence for any HTML/JS page under test:

```text
1. Start live-server (background)
2. Wait 2s
3. Navigate to http://localhost:8080/
4. mcp_playwright_browser_console_messages (level: "error") — fail on JS errors
5. mcp_playwright_browser_snapshot — verify content and layout
6. mcp_playwright_browser_take_screenshot — capture initial state
7. [interact: click, fill, hover as needed]
8. mcp_playwright_browser_snapshot — verify post-interaction state
9. mcp_playwright_browser_take_screenshot — capture post-interaction state
10. Kill live-server
```

---

## Artifacts: Where to Save Output

### Folder layout

| Folder            | Contents                                               | Committed? |
| ----------------- | ------------------------------------------------------ | ---------- |
| `./tests/`        | Unit test scripts only (e.g. `tests.js`)               | ✅ Yes     |
| `./test-artifacts/` | Screenshots, traces, any Playwright output files    | ❌ No      |

**Unit test scripts and test artifacts must be kept in separate folders.**
Never save screenshots to `./tests/`, the repo root, or any other location.

### Screenshot naming

Use descriptive, kebab-case names that reflect the state being captured:

```text
test-artifacts/page-initial.png      ← page before any interaction
test-artifacts/page-revealed.png     ← after revealing hidden content
test-artifacts/page-error.png        ← failure evidence
test-artifacts/mobile-view.png       ← responsive/resized state
```

Pass the filename directly to `mcp_playwright_browser_take_screenshot`:

```text
filename: "test-artifacts/page-initial.png"
```

### `.gitignore` requirements

Every repo using this skill must have these entries in `.gitignore`:

```gitignore
# Playwright MCP server internal state
.playwright-mcp/

# Test artifacts — regenerated on each test run
test-artifacts/*
!test-artifacts/.gitkeep
```

The `test-artifacts/.gitkeep` file tracks the folder in git without committing screenshots.

---

## Failure Evidence

When tests fail, capture evidence before stopping the server:

1. Take a screenshot named `test-artifacts/page-failure.png`
2. Collect console errors with `mcp_playwright_browser_console_messages` (`level: "error"`)
3. Take a snapshot to record the DOM state at failure time
4. Include all of the above in the failure state file (`03-test-failure.md` or equivalent)

---

## Error Handling

| Error                              | Response                                             |
| ---------------------------------- | ---------------------------------------------------- |
| `file://` navigation blocked       | Switch to live-server + `http://localhost:8080/`    |
| Port 8080 already in use           | Use port 8081, update navigate URL                   |
| live-server fails to start         | Check `src/` directory exists; report to orchestrator|
| Playwright MCP unavailable         | Fall back to unit tests only; note limitation in report|
| Console errors on load             | Treat as test failure; capture and report            |
| Screenshot save fails              | Verify `test-artifacts/` folder exists; create if missing|

---

## Checklist for Agents Using This Skill

- [ ] `playwright.test.js` written to `./tests/` mirroring all MCP checks performed
- [ ] `file://` URLs never used — only `http://localhost:8080/`
- [ ] live-server started as background process before navigating
- [ ] 2-second wait applied after server start
- [ ] live-server stopped after all tests complete
- [ ] Screenshots saved to `./test-artifacts/` with descriptive names
- [ ] Unit test scripts saved to `./tests/` (not `./test-artifacts/`)
- [ ] `.gitignore` includes `test-artifacts/*` and `.playwright-mcp/`
- [ ] Console errors checked with `level: "error"`
- [ ] Failure screenshots captured before server teardown (on failure)
