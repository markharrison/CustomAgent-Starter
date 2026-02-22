# Plan: Color Palette JSON Creator Agent

**TL;DR** — A single self-contained **Agent** that reads a palette theme description from the user, generates a set of named, role-tagged colors matching the theme, and writes the result as a JSON file. One-pass, no pipeline needed.

---

## Steps

1. Create `.github/agents/palette-creator.agent.md` using the **Agent** archetype
2. Write YAML frontmatter with model, tools, argument-hint
3. Write `## Context` section (no external skills needed — color knowledge is intrinsic)
4. Write `## DO / DON'T` with clear scope (JSON only, no CSS/images/design assets)
5. Write `## Workflow` with 3 phases: Parse → Generate → Write
6. Write `## Output Files` table documenting the deliverable path
7. Write `## Validation Checklist`

---

## Agent Design

| Field | Value |
|---|---|
| Archetype | Agent |
| Model | `Claude Haiku 4.5` — lightweight utility task |
| Tools | `["edit"]` — only writes the JSON output |
| `user-invocable` | `true` |
| `argument-hint` | `"Describe a palette theme, e.g. 'Sunset palette with 8 colors'"` |

---

## JSON Schema per color entry

```json
{
  "name": "Golden Hour",
  "hex": "#FF9A3C",
  "role": "primary",
  "description": "Warm amber capturing the low sun at dusk"
}
```

**Default roles (5 colors):** `primary`, `secondary`, `accent`, `background`, `text`
**8-color roles:** adds `highlight`, `muted`, `surface`

---

## Workflow Phases

1. **Parse** — extract theme name + color count from prompt; derive kebab-case filename (e.g. `tropical-ocean.json`); default count to 5 if not given
2. **Generate** — produce visually coherent colors matching the theme; assign roles; validate all hex codes are valid `#RRGGBB`; ensure background/text have sufficient contrast
3. **Write** — write JSON array to `./output/palettes/{palette-name}.json`; confirm path and count to user

---

## Deliverable

```
./output/palettes/sunset.json
./output/palettes/tropical-ocean.json
```

---

## Verification

1. Invoke agent with `"Sunset palette"` — should produce 5-entry JSON at `./output/palettes/sunset.json`
2. Invoke with `"Tropical Ocean palette with 8 colors"` — should produce 8-entry JSON with correct roles
3. Confirm all hex values are valid `#RRGGBB`
4. Confirm roles are unique per palette
5. Confirm no CSS, HTML, or non-JSON artifacts are written

---

## Decisions

- Single Agent (not pipeline) — generation + output fits cleanly in one pass
- `Claude Haiku 4.5` — color palette generation is a lightweight creative task; no heavy reasoning needed
- `tools: ["edit"]` only — agent reads nothing (knowledge is intrinsic), only writes the output file
- Dynamic filename derived from the palette name — no hardcoded `palette.json`
- No state/checkpoint file — single-pass agent completes atomically
