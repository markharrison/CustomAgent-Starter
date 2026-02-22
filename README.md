# CustomAgent-Starter

A starter template for creating high-quality custom agent definition files for GitHub Copilot.

This repository provides conventions, templates, and guidance for scaffolding three types of agents: **Agents**, **Orchestrators**, and **Subagents**. Use this to build structured, maintainable agents with clear state management, validation gates, and error handling.

---

## Quick Start

**For detailed guidance**, read [`.github/instructions/custom-agent.instructions.md`](.github/instructions/custom-agent.instructions.md) before creating any agent file.

### Three Agent Archetypes

| Type | Purpose | User-Invocable | Where | Best For |
|------|---------|:-----:|-------|----------|
| **Agent** | Self-contained task | ✅ Yes | `.github/agents/` | Single task that runs end-to-end |
| **Orchestrator** | Coordinates a pipeline | ✅ Yes | `.github/agents/` | Multi-step workflow with gates and validation |
| **Subagent** | Executes one pipeline step | ❌ No | `.github/agents/_subagents/` | Specialist handling one workflow step |

---

## Key Concepts

### Frontmatter
Every agent file starts with YAML frontmatter defining metadata, tools, and archetype-specific settings.

```yaml
---
name: "My Agent"
description: "What it does and what it doesn't do (50-150 chars)"
model: ["Claude Opus 4.6"]
tools: ["execute", "read", "edit"]
user-invocable: true
agents: []
---
```

### State Files vs Deliverables

- **State files** (`./output/state/`) — Progress tracking, decisions, checkpoints
- **Deliverables** (`./output/src/`, `./output/infra/`, etc.) — Actual work product (code, templates, configs)

All agents create a state file at startup with phases unchecked, then update progressively as work completes.

### Workflow Phases
Structure internal work as numbered phases with clear inputs and outputs.

```markdown
## Workflow

### Phase 1: Analyze Input
### Phase 2: Generate Output
### Phase 3: Validate Results
```

### Pipeline & Gates (Orchestrators Only)
Orchestrators define a multi-step pipeline with gates between steps:

- **Approval** gate — human decision required
- **Auto** gate — automatic validation; proceed if valid
- **—** gate — no gate; used for final steps

---

## File Structure

```
.github/
├── instructions/
│   └── custom-agent.instructions.md       ← READ THIS FIRST
├── agents/
│   ├── my-agent.agent.md                  ← User-invocable agent
│   ├── my-orchestrator.agent.md           ← User-invocable orchestrator
│   └── _subagents/
│       ├── step-1-analyzer.agent.md       ← Subagent (not user-invocable)
│       ├── step-2-generator.agent.md
│       └── step-3-validator.agent.md
└── skills/
    └── {domain}/
        └── SKILL.md                        ← Domain knowledge & templates
```

---

## Getting Started

### 1. Choose Your Archetype
Decide: Is this a **single task** (Agent), a **multi-step workflow** (Orchestrator), or a **pipeline step** (Subagent)?

### 2. Read the Instructions
Study the relevant sections in [`.github/instructions/custom-agent.instructions.md`](.github/instructions/custom-agent.instructions.md):
- **Frontmatter** — required fields for your archetype
- **Body Structure** — sections to include/exclude
- **State Management** — how to track progress and enable resume/revert
- **Validation Checklist** — quality gates

### 3. Create Your Agent File
- Save to `.github/agents/{name}.agent.md` or `.github/agents/_subagents/{name}.agent.md`
- Use your archetype's frontmatter template
- Follow the body structure guidelines
- Include all mandatory sections

### 4. Validate
Before publishing:
- [ ] Frontmatter is valid YAML
- [ ] All required fields present
- [ ] `## DO / DON'T` section present with scope boundaries
- [ ] State file strategy documented
- [ ] `## Validation Checklist` present
- [ ] No copy/paste errors in templates

---

## Common Patterns

### Agent with State Checkpoints
```markdown
## DO / DON'T

### DO
- ✅ STATE FILE REQUIRED: Create `./output/state/00-{agent-name}.md` at startup
- ✅ Update state file after each phase — check off with inline summary

### DON'T
- ❌ Write state file only at the end — create it first for visibility
```

### Orchestrator with Pipeline & Gates
```markdown
## Pipeline Definition

| Step | Subagent | State File | Deliverables | Gate | On Fail |
|------|----------|------------|--------------|------|---------|
| 1 | Analyzer | `01-analysis.md` | — | Approval | Stop |
| 2 | Generator | `02-code.md` | `./output/src/*` | Auto | Retry once |
| 3 | Validator | `03-tests.md` | `./output/tests/*` | Approval | Back to Step 2 |
```

### Subagent with Prerequisites & Workflow
```markdown
## Prerequisites Check
- `01-analysis.md` — previous step's analysis results

## Workflow
### Phase 1: Read Input
### Phase 2: Generate Deliverable
### Phase 3: Validate
```

---

## Key Rules

✅ **DO:**
- Create state files at startup (before Phase 1), not at the end
- Update state files progressively after each phase
- Separate state files (`./output/state/`) from deliverables (`./output/src/`, etc.)
- Document scope and boundaries in `## DO / DON'T`
- Use explicit agent names in `agents:` field (principle of least privilege)
- Delegate all work in orchestrators (never do direct work)

❌ **DON'T:**
- Write state file only at the end — create it first for live visibility
- Put code/templates in `./output/state/` — that's for tracking only
- Use handoffs in orchestrated pipelines — use `#runSubagent` instead
- Skip gates or auto-approve in orchestrators
- Exceed archetype boundaries (e.g., subagent doing work outside its step)

---

## References

- **Detailed Instructions**: [`.github/instructions/custom-agent.instructions.md`](.github/instructions/custom-agent.instructions.md)
- **Official GitHub Docs**: https://docs.github.com/en/copilot/how-tos/use-copilot-agents
- **Current Model Recommendations**: See instructions → "Model Selection"
- **Tool Configuration**: See instructions → "Tool Configuration"

---

## License

See [LICENSE](LICENSE) for details.