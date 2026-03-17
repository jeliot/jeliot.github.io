# Technical Overview: Focused Graph Analysis Report Generation

This document describes how the Focused Graph Analysis report is generated in `focused-analysis.html`. The report analyzes Desmos calculator graph state to produce performance diagnostics and dependency insights.

---

## 1. Architecture Overview

The report generation follows a **pipeline architecture**:

```
Input (URL or JSON) → State Fetch → analyze() → Analysis Object → genReport() → HTML Output
```

- **Input**: Desmos graph URL or pasted graph state JSON
- **State**: Desmos `getState()` format (`state.expressions.list`, etc.)
- **Analysis**: Single `analyze(state)` call produces an analysis object
- **Report**: `genReport(a, graphUrl)` concatenates section HTML

All logic runs client-side in a single HTML file. LaTeX rendering uses KaTeX.

---

## 2. Input Processing

### 2.1 State Acquisition

- **URL**: `fetchState(hash)` fetches `https://www.desmos.com/calculator/{hash}` with `Accept: application/json`
- **Paste**: User pastes raw JSON; `JSON.parse()` extracts `state`
- **Structure**: `state.expressions.list` is the primary array of items (expressions, folders, images, tables)

### 2.2 Item Types

Items in `state.expressions.list` include:

- `expression` — LaTeX expressions, sliders, points
- `folder` — Folders containing expressions
- `image` — Image expressions
- `table` — Table expressions
- `text` — Notes

Only `expression` items are parsed for the dependency and cost analysis. Folders, images, and tables are counted and surfaced in specific sections.

---

## 3. LaTeX Parsing

### 3.1 Core Functions

| Function | Purpose |
|----------|---------|
| `parseExpr(latex)` | Parse a single expression; returns `{type, name, refs, isFunc, isRec, isImplicit, params, rhs, calls}` |
| `varTokens(latex)` | Extract variable identifiers (subscripts, point refs, single letters) |
| `findFnCalls(latex)` | Extract function calls with arguments |
| `findDefEq(latex)` | Find top-level `=` for assignment split |
| `findDefIneq(latex)` | Find inequality operators for implicit-style expressions |

### 3.2 Expression Types

- **function**: `f(x)=...` — LHS matches `name\left(` pattern
- **implicit**: `y=...` or `x=...` — name is `y`/`x` or refs only `x`/`y`
- **variable**: Named assignment, e.g. `a=5`
- **display**: No assignment; value shown inline (e.g. `5`, `f(3)`)
- **action**: Contains `->` or `\to` (Computation Layer)

### 3.3 Recursion Detection

`isRec` is set when the RHS contains a call to the function’s own name (e.g. `f(x)=f(x-1)+1`).

### 3.4 Boolean Variables

`isBooleanVariable(e)` is true when the RHS is **only** curly-bracket segments (`\left\{...\right\}`). Handled by `rhsIsOnlyCurlyBracketSegments(rhs)`.

---

## 4. Dependency Graph

### 4.1 Structure (`buildDeps(parsedList)`)

| Property | Type | Description |
|----------|------|-------------|
| `def` | `{name → parsed}` | Definitions by name |
| `on` | `{name → Set<dep>}` | Direct dependencies (refs + called functions) |
| `by` | `{name → Set<dep>}` | Direct dependents |
| `all` | `Set<name>` | All defined names |
| `callsTo` | `{name → [{fn, args, argVars}]}` | Outgoing function calls |
| `calledBy` | `{fn → [{caller, args}]}` | Incoming callers |
| `trans(n)` | Function | Transitive downstream (all dependents) |
| `transUp(n)` | Function | Transitive upstream (all dependencies) |

### 4.2 Dependency Sources

- **Variable refs**: From `varTokens(rhs)` — variables used in the RHS
- **Function calls**: From `findFnCalls(rhs)` — functions invoked
- Only **named** expressions contribute to the graph; display expressions do not

---

## 5. Cost Estimation

### 5.1 `eCost(p)`

Heuristic cost from expression structure:

- Base: 1 + `0.003 * latex.length`
- Recursive: +30
- Summation: +6
- Integral: +10
- Sum + integral: +20
- `\operatorname{for}`: +8
- `\operatorname{repeat}`: +6
- `\operatorname{polygon}`: +4
- `\operatorname{with}` / `\operatorname{join}`: +2
- `\left\{`: +1

### 5.2 Severity Tiers

`costSeverity(cost)` maps to: `critical` (>20), `high` (>8), `medium` (>3), `low` (≤3).

---

## 6. Analysis Object

`analyze(state)` returns:

| Property | Description |
|----------|-------------|
| `m` | Counts: expr, fold, img, slider, drag, movPt, rec, fn, etc. |
| `folders` | `{id → {title, hidden}}` |
| `exprs` | Enriched expressions: `{id, latex, hidden, slider, drag, p, line, color, ...}` |
| `parsed` | Parsed results aligned with `exprs` |
| `dep` | Dependency graph |
| `usedByStyle` | `Set` of names referenced in style properties (color, label, etc.) |
| `usedByPolygon` | `Set` of names used in polygon/display expressions (with fill/opacity/width) |
| `usedByLabel` | `Set` of names used in expressions with enabled labels (incl. label text) |
| `interactables` | Sliders, draggables, movable points with downstream cost |
| `interactiveDownstream` | Names that depend on interactables |
| `inputs` | Root variables (no dependencies) with affected downstream |
| `lineMap` | `{name → line}` for navigation |
| `items`, `allItemsWithLine` | Raw items with line numbers |

---

## 7. “Used By” Sets

These sets mark variables as used so they are not reported as unused:

### 7.1 `usedByStyle`

- Built by `buildUsedByStyle(items, dep)`
- Recursively scans all items for variable names in string properties
- Skips `latex` when long, hex colors, and numeric strings
- Uses `normalizeVarForMatch()` for `a_1` ↔ `a_{1}`

### 7.2 `usedByPolygon`

- Expressions with `hasFillOrOpacity(e)` (fill, fillOpacity, lineOpacity, pointOpacity, lineWidth)
- And either `hasPolygon(latex)` or `type === 'display'` (non-numeric)
- Collects `refs` and `calls` from those expressions

### 7.3 `usedByLabel`

- Expressions with `showLabel === true`
- Collects `refs`, `calls`, and `extractLabelRefs(e.label, dep)`
- `extractLabelRefs` parses `${var}`, LaTeX tokens, and function calls in label text

---

## 8. Report Sections (Order)

1. **Heavy Expressions** — Cost ≥ 4, non-polygon; classified by impact (loading vs interactivity)
2. **Reusable Patterns** — Sub-expressions repeated across expressions without a named variable
3. **Interactive Dependencies** — Sliders, draggables, movable points and their downstream
4. **Implicit Expressions and Regressions** — `y=`, `x=`, or `~` expressions
5. **Polygons & Graph Objects** — Display with fill/opacity/width, or polygon in LaTeX
6. **Images** — Image expressions
7. **Labels** — Expressions with `showLabel`
8. **Custom Functions and Booleans** — Functions and boolean variables
9. **Actions** — `->` / `\to` expressions
10. **Unused Expressions** — Named definitions without references (with disclaimer)
11. **Unstored Values** — Inline function calls and numerical displays
12. **Dependency Concept Map** — Layered dependency graph
13. **Other Items** — Items not in any section above

---

## 9. Section Classification Logic

### 9.1 Heavy Expressions

- `eCost(p) >= COST_THRESHOLD` (4)
- Excludes `j_{uice}`, polygon expressions
- `classifyHeavy()` assigns impact (load, interact, both) from `interactiveDownstream`

### 9.2 Polygons

- `hasFillOrOpacity(e)` (fill, fillOpacity, lineOpacity, pointOpacity, lineWidth)
- And either `hasPolygon(latex)` or `(type === 'display' && !isNumericalConstant(latex))`
- Excludes expressions with `showLabel`

### 9.3 Labels

- `e.showLabel === true`

### 9.4 Custom Functions and Booleans

- `p.isFunc` or `isBooleanVariable(e)`

### 9.5 Unused

- Named, not action, not implicit, not polygon, not used by style/polygon/label
- Excludes functions, booleans, labels, sliders, draggables, movable points
- Excludes `dep.by[name]` or `calledBy[name]` non-empty

### 9.6 Unstored Values

- Function calls not stored in a variable (e.g. `f(x)` in `y = f(x) + 1`)
- Numerical display expressions (e.g. `5`, `\pi`)
- Display expressions with function calls (e.g. `f(5)`)
- Excludes implicit, action, recursive, polygon, display (except numerical and display-with-calls)

### 9.7 Other Items

- `findOtherItems(a)` builds `exprIdsInSection` from all sections
- Items not in that set are “Other”

---

## 10. HTML Generation

### 10.1 `genReport(a, graphUrl)`

1. Build TOC sidebar
2. Stats grid from section counts
3. Call each `gen*Section(a)` in order
4. Assemble footer

### 10.2 LaTeX Rendering

- `latexEl(tex, line)` produces `<span data-latex="..." data-line="...">`
- `renderAllLatex(root)` runs KaTeX on `[data-latex]` elements
- Fallback text: "Nameless expression" on render failure

### 10.3 Concept Map Interaction

- `drawConceptMap()` draws SVG edges between nodes
- `selectNode(name)` highlights upstream/downstream
- `showNodeDetail(name)` shows node details in a side panel

---

## 11. Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `COST_THRESHOLD` | 4 | Heavy expression cutoff |
| `COST_CRITICAL` | 20 | Critical severity |
| `COST_HIGH` | 8 | High severity |
| `COST_MEDIUM` | 3 | Medium severity |

---

## 12. File Structure

`focused-analysis.html` contains:

- **CSS** (lines ~1–250): Theming, layout, components
- **HTML** (lines ~250–300): Input form, container
- **JavaScript** (lines ~300–1840): Parser, analysis, report generation, interaction

No external JS libraries except KaTeX for LaTeX rendering.
