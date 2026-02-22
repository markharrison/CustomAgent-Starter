# JS Unit Testing Skill

Conventions and patterns for writing unit test scripts in agent pipelines.
Read this skill before writing any unit test file.

---

## Core Principles

- **No external dependencies** — use Node.js built-ins only (`fs`, `path`)
- **Run with plain Node** — `node tests/tests.js` — no test runner needed
- **Static analysis** — read source files as text; use regex and `includes()` to assert
- **Exit codes signal pass/fail** — `process.exit(1)` on any failure so CI catches it
- **Self-contained harness** — define `assert()` and `section()` inline; no imports

---

## File Location and Naming

| File          | Location    | Run command             |
| ------------- | ----------- | ----------------------- |
| `tests.js`    | `./tests/`  | `node tests/tests.js`   |

Always use `tests/tests.js`. Do not create additional unit test files or subfolders
unless the pipeline explicitly requires it (e.g. separate test suites per deliverable).

---

## Harness Template

Every `tests.js` starts with this exact harness — do not vary the structure:

```js
/**
 * tests.js — {description of what is being tested}
 * No external dependencies required — uses Node.js built-ins only.
 * Run with: node tests/tests.js
 */

"use strict";

const fs   = require("fs");
const path = require("path");

// ── Test harness ─────────────────────────────────────────────────────────────

let passed = 0;
let failed = 0;
const failures = [];

function assert(description, condition, detail) {
  if (condition) {
    console.log(`  ✅ ${description}`);
    passed++;
  } else {
    console.error(`  ❌ ${description}${detail ? ` — ${detail}` : ""}`);
    failed++;
    failures.push({ description, detail });
  }
}

function section(title) {
  console.log(`\n── ${title} ──`);
}
```

---

## Loading Source Files

Resolve paths relative to `__dirname` so the script works regardless of cwd:

```js
const SRC_DIR = path.resolve(__dirname, "../src");

section("Loading source files");

const htmlPath = path.join(SRC_DIR, "index.html");
const cssPath  = path.join(SRC_DIR, "style.css");
const jsPath   = path.join(SRC_DIR, "script.js");

assert("index.html exists", fs.existsSync(htmlPath));
assert("style.css exists",  fs.existsSync(cssPath));
assert("script.js exists",  fs.existsSync(jsPath));

// Guard: only read if file exists — prevents crash on missing file
const html = fs.existsSync(htmlPath) ? fs.readFileSync(htmlPath, "utf8") : "";
const css  = fs.existsSync(cssPath)  ? fs.readFileSync(cssPath,  "utf8") : "";
const js   = fs.existsSync(jsPath)   ? fs.readFileSync(jsPath,   "utf8") : "";
```

The guarded read pattern (`condition ? readFileSync : ""`) is mandatory — it prevents
all subsequent `assert()` calls from throwing when a file is missing.

---

## Assertion Patterns

### Exact string presence

```js
assert("setup text present in HTML",
  html.includes("expected text here"),
  'Expected: "expected text here"');
```

### Regex match

```js
assert("DOCTYPE declaration present",
  /<!DOCTYPE\s+html>/i.test(html));

assert("<html> element with lang attribute",
  /<html\s[^>]*lang=/i.test(html),
  "Accessibility: lang attribute required");
```

### File linkage

```js
// CSS link — attribute order-independent
assert("CSS file linked via <link rel=\"stylesheet\">",
  /<link\s[^>]*rel=["']stylesheet["'][^>]*href=["'][^"']*style\.css["']/i.test(html) ||
  /<link\s[^>]*href=["'][^"']*style\.css["'][^>]*rel=["']stylesheet["']/i.test(html),
  "style.css must be referenced in <head>");

// Script tag
assert("script.js linked via <script src>",
  /<script\s[^>]*src=["'][^"']*script\.js["']/i.test(html));
```

### Non-empty file

```js
assert("CSS file is non-empty",
  css.trim().length > 0);
```

### JS static analysis

```js
assert("click event listener attached",
  js.includes("addEventListener") && js.includes("click"));

assert("aria-expanded toggled in JS",
  js.includes("aria-expanded"));
```

---

## Standard Section Order

Use this ordering for web page tests — add/remove sections as the deliverable requires:

| # | Section heading            | What it checks                                      |
| - | -------------------------- | --------------------------------------------------- |
| 1 | Loading source files        | Files exist, load without error                     |
| 2 | HTML structure              | DOCTYPE, `<html lang>`, `<head>`, `<body>`, charset, viewport, `<title>` |
| 3 | Content                     | Expected text/data from state file is present in HTML |
| 4 | CSS linkage                 | `<link rel="stylesheet">` present, file non-empty   |
| 5 | JS linkage                  | `<script src>` present, file non-empty              |
| 6 | Accessibility and semantics | `<main>`, `<h1>`, `<button>`, `aria-*` attributes   |
| 7 | JavaScript static analysis  | Key IDs, event listeners, class/attribute toggles   |

---

## Results Footer

Every `tests.js` ends with this exact footer — do not vary it:

```js
// ── Results ──────────────────────────────────────────────────────────────────

const total = passed + failed;
console.log(`\n${"─".repeat(50)}`);
console.log(`Results: ${passed}/${total} passed, ${failed} failed`);

if (failed > 0) {
  console.error("\nFailing checks:");
  failures.forEach(f => console.error(`  ❌ ${f.description}${f.detail ? ` — ${f.detail}` : ""}`));
  process.exit(1);
} else {
  console.log("All tests passed ✅");
  process.exit(0);
}
```

`process.exit(1)` on failure is mandatory — it signals failure to the orchestrator
and to any CI runner.

---

## Content Assertions — Sourced from State Files

When testing that content matches a state file (e.g. joke text from `01-joke.md`),
read the state file and extract the expected strings — do not hardcode them:

```js
// Read expected content from state file
const jokePath = path.resolve(__dirname, "../output/state/01-joke.md");
const jokeText = fs.existsSync(jokePath) ? fs.readFileSync(jokePath, "utf8") : "";

// Extract setup and punchline (adjust regex to match state file format)
const setupMatch    = jokeText.match(/\*\*Setup\*\*[:\s]+(.+)/);
const punchlineMatch = jokeText.match(/\*\*Punchline\*\*[:\s]+(.+)/);

if (setupMatch) {
  assert("Joke setup present in HTML",
    html.includes(setupMatch[1].trim()),
    `Expected: "${setupMatch[1].trim()}"`);
}
```

This keeps tests in sync with the actual generated content rather than brittle hardcoded strings.

---

## Checklist for Agents Using This Skill

- [ ] Harness copied exactly — `assert()`, `section()`, `passed`/`failed`/`failures` vars
- [ ] All paths resolved via `path.resolve(__dirname, ...)` — not relative strings
- [ ] Guarded reads used — files read only after existence check
- [ ] Content assertions sourced from state files — not hardcoded
- [ ] Standard section order followed
- [ ] Results footer present and unmodified
- [ ] `process.exit(1)` on failure, `process.exit(0)` on pass
- [ ] File saved to `./tests/tests.js`
- [ ] Verified runnable: `node tests/tests.js` exits 0 on pass, 1 on fail
