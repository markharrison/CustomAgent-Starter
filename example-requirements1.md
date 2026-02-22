# Plan: Joke Orchestrator Custom Agents

Create an Orchestrator + 3 Subagents for a joke-to-webpage pipeline: **Joker** (generates animal joke) → **Coder** (builds HTML/JS/CSS page) → **Tester** (validates the page). Joker→Coder has an Approval gate; Coder→Tester is Auto. Follows the Coder/Tester bounce-back pattern from the agent instruction guide.

---

## Files to Create

| # | File | Archetype |
|---|------|-----------|
| 1 | `.github/agents/joke-pipeline.agent.md` | Orchestrator |
| 2 | `.github/agents/_subagents/joker.agent.md` | Subagent (Step 1) |
| 3 | `.github/agents/_subagents/coder.agent.md` | Subagent (Step 2) |
| 4 | `.github/agents/_subagents/tester.agent.md` | Subagent (Step 3) |

## Pipeline Design

| Step | Subagent | State File | Deliverables | Gate | On Fail |
|------|----------|------------|--------------|------|---------|
| 1 | Joker | `01-joke.md` | — | Approval | Stop |
| 2 | Coder | `02-code.md` | `./output/src/index.html`, `./output/src/style.css`, `./output/src/script.js` | Auto | Retry once |
| 3 | Tester | `03-test.md` | `./output/tests/tests.js` (unit), `./output/tests/playwright.test.js` (functional), `./output/test-artifacts/*.png` (screenshots, gitignored) | — | Back to Step 2 (Coder) |

---

## Phase 1: Create Subagents (parallel — no dependencies between them)

### Step 1a — Joker subagent (`.github/agents/_subagents/joker.agent.md`)

- Model: see **Model Selection** table
- Tools: `["read", "edit"]`
- Generates a joke about animal(s) based on orchestrator prompt
- Writes `01-joke.md` to `./output/state/` with the joke text
- No deliverables — joke travels via state file

### Step 1b — Coder subagent (`.github/agents/_subagents/coder.agent.md`)

- Model: see **Model Selection** table
- Tools: `["read", "edit", "search"]`
- Reads joke from `01-joke.md`, builds an HTML/CSS/JS page in `./output/src/`
- Prerequisite: `01-joke.md` must exist
- On bounce-back from Tester: reads `03-test-failure.md` for failure details and fixes code

### Step 1c — Tester subagent (`.github/agents/_subagents/tester.agent.md`)

- Model: see **Model Selection** table
- Tools: `["read", "edit", "search", "execute", "playwright/*"]`
- Reads both skills before starting any work:
  - **`.github/skills/js-unit-testing/SKILL.md`** — harness template, assertion patterns, file location, exit codes
  - **`.github/skills/playwright-testing/SKILL.md`** — live-server setup, `http://` requirement, artifact folder layout, tool reference, persisting tests as a script
- **Unit Tests**: Writes `./output/tests/tests.js` following the `js-unit-testing` skill
- **Functional Tests**: Runs Playwright MCP checks following the `playwright-testing` skill;
  writes `./output/tests/playwright.test.js` so tests are replayable outside the agent session
- Prerequisite: `02-code.md` and `./src/` files must exist
- On fail: writes `03-test-failure.md` with failing checks; orchestrator bounces back to Coder (max 1 bounce, then Stop)

## Phase 2: Create Orchestrator (depends on Phase 1)

### Step 2 — Orchestrator (`.github/agents/joke-pipeline.agent.md`)

- Model: see **Model Selection** table
- Tools: `["read", "search", "edit", "execute", "agent", "playwright/*"]` — union of all subagent tools
- `agents: ["*"]`, `argument-hint: "Describe what kind of animal joke you want"`
- Manages `00-{orchestrator-name}.md` state file, gate decisions, resume/revert/reset
- Delegates ALL work — never creates jokes or code directly

---

## Model Selection

| Agent | Model | Rationale |
|-------|-------|-----------|
| Joke Pipeline | Claude Sonnet 4.6 | Coordination and gate management need strong reasoning |
| Joker | Claude Sonnet 4.6 | Creative generation, balanced speed/quality |
| Coder | Claude Sonnet 4.6 | Code generation — speed matters |
| Tester | Claude Sonnet 4.6 | Validation/analysis — balanced |

## Verification

1. All 4 files have valid YAML frontmatter
2. Subagents: `user-invocable: false`, `agents: []`
3. Orchestrator: `user-invocable: true`, `agents: ["*"]`, tools superset of all subagents
4. Gate types match requirements: Joker→Coder = `Approval`, Coder→Tester = `Auto`
5. Each subagent has required sections: Prerequisites Check, Workflow, Output Files, DO/DON'T, Validation Checklist
6. Orchestrator has required sections: Pipeline Definition, State Management, Gate Model, Subagent Delegation, DO/DON'T, Validation Checklist
7. Coder/Tester bounce-back pattern configured correctly (max 1 bounce)

## Decisions

- **Skills used**: `js-unit-testing` and `playwright-testing` skills in `.github/skills/` — Tester reads both before starting work
- **Code deliverables in `./output/src/`** — separate HTML, CSS, JS files (not inline)
- **Test deliverables in `./output/tests/`** — unit test scripts only
- **Playwright artifacts in `./output/test-artifacts/`** — screenshots and traces (gitignored, regenerated each run); kept separate from test scripts
- **No deliverables in repo root** — all agent output goes to `./output/` subfolders
- **Single-bounce limit** on Tester→Coder loop, then Stop
