# Project Pulse Dashboard — Implementation Plan

## 1. Summary

**Project Pulse** is a small, dependency-free static web app that renders a set of project cards from a local JSON file. Its purpose is to let Mona's contributors scan, at a glance, what is active, who owns it, what state it is in, what changed recently, and how urgent it is.

**Goals**
- Render a polished, card-based dashboard with the exact title **"Project Pulse"**.
- Load project data from `app/project-data.json` (top-level `projects` array) and render one `.project-card` per project.
- Each card shows `name`, `owner`, `status`, `recentActivity`, `priority`, and a progress indicator.
- Ship a VS Code launch configuration named **"Run Project Pulse Dashboard"** that serves `app/` on port 5500 and opens `http://localhost:5500/index.html` (never a directory listing).
- No frameworks, no build step, no runtime deps — HTML + CSS + a single `fetch` in vanilla JS.
- Match conventions already encoded in `.github/agents/*.md`, `.github/steps/3-step.md`, and `scripts/validate-exercise.sh` (deterministic selectors, keyphrases, file layout, strict-JSON launch config).

**Non-goals:** auth, editing, persistence, backend, framework, filtering/sorting (v2).

---

## 2. Ordered implementation steps

### Step 1 — Freeze the data contract and write sample data
Establish the schema so Designer and Coder can work in parallel from a locked spec.

- Top-level object: `{ "projects": [ ... ] }`.
- Each project object fields:
  - `name` (string, required)
  - `owner` (string, required)
  - `status` (enum: `"On Track"`, `"At Risk"`, `"Blocked"`, `"Completed"`)
  - `recentActivity` (string, required)
  - `priority` (enum: `"Low"`, `"Medium"`, `"High"`, `"Critical"`)
  - `progress` (integer 0–100, optional)
  - `summary` (string, optional — the "contributor-friendly blurb" from Mona's brief)
- Produce 5–6 realistic sample projects that collectively cover every `status` value and every `priority` value, and include: one long `name`, one long `recentActivity`, one `progress: 0`, one `progress: 100`, one entry with `summary` present and one without.

**File touched:** `app/project-data.json` — **Coder (sole writer).**

### Step 2 — Design system + CSS
Establish tokens, deterministic hooks, and responsive rules.

- CSS custom properties for color, spacing, radius, shadow, typography.
- Required deterministic selectors (workflow-checked keyphrases):
  - `.dashboard` (page container)
  - `.project-card` (individual card)
  - Must include `border-radius` and `box-shadow` declarations somewhere in this file.
- Supporting hooks (Designer's judgment, but recommended for clarity):
  - `.dashboard-header`, `.project-grid`
  - `.status-badge`, `.status-badge--on-track`, `.status-badge--at-risk`, `.status-badge--blocked`, `.status-badge--completed`
  - `.priority`, `.priority--low|--medium|--high|--critical`
  - `.progress`, `.progress__bar`
  - `.empty-state`, `.error-state`
- Responsive: `grid-template-columns: repeat(auto-fill, minmax(280px, 1fr))`, collapse to one column below ~480px.
- Accessibility: WCAG AA contrast on badges, visible focus outline, `prefers-reduced-motion` guard; do not encode status by color alone (pair with text label).

**File touched:** `app/styles.css` — **Designer (sole writer).**

### Step 3 — HTML shell and semantic markup
Structure the page and expose class hooks the CSS and JS depend on.

- Doctype, `<html lang="en">`, charset, viewport meta.
- `<title>Project Pulse</title>` and a visible `<h1>Project Pulse</h1>` (exact string required).
- `<link rel="stylesheet" href="styles.css">` (relative — served from `app/`).
- Semantic layout:
  - `<header class="dashboard-header">` with title and short subtitle.
  - `<main class="dashboard">` wrapping content.
  - `<section class="project-grid" id="project-grid" aria-live="polite" aria-busy="true">` — render target.
  - `<template id="project-card-template">` containing the `.project-card` markup with slots for name, owner, status badge, recent activity, priority, progress, and optional summary.
  - Hidden `.empty-state` and `.error-state` nodes.
- `<noscript>` fallback message.
- The file must contain textual references to `project-data.json` (used by the fetch call) and `styles.css` — both required by workflow keyphrase checks.

**File touched:** `app/index.html` — **Designer proposes template/markup; Coder is the sole writer** of the file (see §5 for the single-writer rule).

### Step 4 — Data loading and render logic
Wire the JSON into the DOM.

- Inline `<script defer>` in `app/index.html` (no external JS file — keeps the deliverable to the three specified app files).
- On `DOMContentLoaded`:
  1. `fetch('./project-data.json')`.
  2. Validate `response.ok`, then parse; assert `data.projects` is an array.
  3. If array empty → show `.empty-state`, hide grid.
  4. Otherwise clone `<template id="project-card-template">` per project, populate every field via `textContent` (never `innerHTML`).
  5. Map `status` → badge modifier class through a lookup object; unknown value → neutral badge and `console.warn`.
  6. Map `priority` similarly; missing priority renders as "Unknown".
  7. Clamp `progress` into `[0, 100]`; missing/NaN → hide the progress bar (do not render 0% by accident).
  8. Wrap each per-card render in try/catch so one bad record does not blank the grid.
  9. Set `aria-busy="false"` when done.
- On fetch/parse failure → show `.error-state`, log underlying error, keep the page usable.

**File touched:** `app/index.html` — **Coder (sole writer).**

### Step 5 — VS Code launch configuration
Deterministic run experience.

- `.vscode/launch.json` — **strict JSON, no comments, no trailing commas** (required by Coder agent and by `scripts/validate-exercise.sh`).
- Single configuration:
  - `"name": "Run Project Pulse Dashboard"` (exact — workflow-checked).
  - `"type": "node-terminal"`, `"request": "launch"`.
  - `"command": "python3 -m http.server 5500"` (exact substring required).
  - `"cwd": "${workspaceFolder}/app"` (required — Coder agent instruction).
  - `"serverReadyAction": { "pattern": "Serving HTTP on .* port ([0-9]+)", "uriFormat": "http://localhost:%s/index.html", "action": "openExternally" }` (the URI format substring is workflow-checked).
- Validate: `python3 -m json.tool .vscode/launch.json`.

**File touched:** `.vscode/launch.json` — **Coder (sole writer).**

### Step 6 — Integration validation
Non-file step. Coordinated by the Orchestrator; both agents report.

- Manual: Run and Debug → **Run Project Pulse Dashboard** → browser opens `http://localhost:5500/index.html`, dashboard renders.
- Automated sanity: JSON parses for both JSON files; all keyphrases in §7 present.

---

## 3. File assignments (explicit ownership)

| Step | Required file | Owner (writer) | Contributing role | Notes |
|---|---|---|---|---|
| 1 | `app/project-data.json` | **Coder** | — | Sole writer. Schema + enumerations + sample data. |
| 2 | `app/styles.css` | **Designer** | — | Sole writer. All CSS, tokens, badges, responsive rules, `border-radius`, `box-shadow`. |
| 3 | `app/index.html` (markup/structure) | **Coder** | **Designer** (supplies template snippet + class hooks) | Coder is the only writer to prevent conflicts. Designer delivers markup as a text snippet Coder pastes in. |
| 4 | `app/index.html` (fetch + render script) | **Coder** | — | Same file, same writer. |
| 5 | `.vscode/launch.json` | **Coder** | — | Strict JSON. |
| 6 | — | Orchestrator | Designer + Coder | Verification only, no file writes. |

Note: `docs/project-pulse-plan.md` (this plan) is Planner output and is **not** one of the four required deliverables.

---

## 4. Dependencies between steps

- Step 3 depends on **Step 1** (field names must match JSON keys) and **Step 2** (class hooks must match CSS selectors).
- Step 4 depends on **Step 1** (schema) and **Step 3** (template IDs and DOM structure).
- Step 5 is content-independent; only depends on `app/index.html` existing at *runtime*, so it can start any time after Step 1.
- Step 6 depends on Steps 1–5 all being complete.

The critical shared contract is the JSON schema + enumerations produced in Step 1. Locking it early is what enables parallel work in Phase B.

---

## 5. Parallel vs. sequential (Orchestrator phase guidance)

**Phase A — sequential, blocking (must run first):**
- Step 1 (Coder → `app/project-data.json`). Publish the field list and status/priority enum values to Designer before Phase B.

**Phase B — parallel (disjoint files, disjoint concerns):**
- Step 2 — Designer writes `app/styles.css`.
- Step 5 — Coder writes `.vscode/launch.json`.
- These touch different files with no shared state; run simultaneously.

**Phase C — sequential, single-writer:**
- Step 3 then Step 4, both writing to `app/index.html`. **Coder is the sole writer.** Designer delivers the template markup and class-hook list as a text snippet (not a direct edit) at the *start* of Phase C; Coder integrates it, then adds the fetch/render script.
- This single-writer rule is the key rule the Orchestrator must enforce — `app/index.html` is the only file two roles have opinions on.

**Phase D — sequential:**
- Step 6 after Phases A, B, C are all green.

---

## 6. Edge cases to handle

**Data**
- `project-data.json` missing / non-200 / network error → render `.error-state`, log error.
- Invalid JSON → catch parse exception, render `.error-state`.
- Top-level not an object, or `projects` not an array → treat as error.
- `projects: []` → render `.empty-state` ("No projects yet"), not a blank grid.
- Individual record missing required fields → render card with placeholder (e.g., `—`); do not crash the whole loop.
- Unknown `status` / `priority` value → neutral badge + `console.warn`.
- `progress` missing / NaN / <0 / >100 → clamp or hide the progress bar.
- Any field containing HTML/script → `textContent` only (XSS defense).

**Content / UX**
- Very long project names → `overflow-wrap: anywhere` and 2-line clamp with `title` attribute for full text.
- Long `recentActivity` → clamp to 3 lines with ellipsis; full text in `title`.
- Many projects (50+) → pure CSS grid stays performant; no virtualization needed.
- Small screens (<480px) → single column, badges wrap under title, tap targets remain ≥ 24px.
- `prefers-reduced-motion: reduce` → disable progress-bar / hover transitions.
- Do not encode status by color alone; always include the text label.

**Runtime / launch**
- Port 5500 already in use → Python raises; learner restarts. Note in troubleshooting.
- Opening `app/index.html` via `file://` blocks `fetch` in most browsers → the launch config's `http://localhost` URL is the supported path; do not encourage double-clicking the file.
- `serverReadyAction` regex must match Python's actual line `Serving HTTP on 0.0.0.0 port 5500 (...)`; the pattern in Step 5 captures the port group.
- `.vscode/launch.json` must be strict JSON — no `//` comments, no trailing commas — or VS Code silently ignores the config and `scripts/validate-exercise.sh` fails on `python3 -m json.tool`.

---

## 7. Validation expectations

**Automated / mechanical**
- `python3 -m json.tool app/project-data.json` → exits 0.
- `python3 -m json.tool .vscode/launch.json` → exits 0.
- Keyphrases present (matches `.github/workflows/3-step.yml` checks):
  - `app/index.html` contains: `Project Pulse`, `styles.css`, `project-data.json`, `project-card`, `index.html`-target reference, `name`, `recentActivity`, `priority`, `status`.
  - `app/styles.css` contains: `.dashboard`, `.project-card`, `border-radius`, `box-shadow`.
  - `app/project-data.json` contains: `projects`, `name`, `owner`, `status`, `recentActivity`, `priority`.
  - `.vscode/launch.json` contains: `Run Project Pulse Dashboard`, `${workspaceFolder}/app`, `python3 -m http.server`, `http://localhost:%s/index.html`.

**Manual**
- Run and Debug → **Run Project Pulse Dashboard** → green play.
- Browser opens at `http://localhost:5500/index.html` (not a directory listing).
- Visible `<h1>Project Pulse</h1>`.
- ≥ 4 cards render; each shows name, owner, status badge, recent activity, priority, progress.
- Status badges visually distinct across all four statuses; priority treatment distinct across all four levels.
- Resize to ~375px → grid collapses to one column, no horizontal scroll.
- Temporarily rename `project-data.json` → refresh → `.error-state` visible (not blank).
- Temporarily set `"projects": []` → refresh → `.empty-state` visible.
- Quick a11y pass: badge contrast passes AA; visible focus ring on any focusable element; no color-only status encoding.
- Stop the debugger; confirm port 5500 releases.

---

## 8. Open questions

1. **Interactivity scope** — Are cards purely informational, or do they link to a project URL? Recommend informational only for v1; allow optional `url` field in JSON but do not require it.
2. **Progress field** — Should we include `progress` (0–100) and render a bar? Recommend yes (matches the user's stated requirements); confirm with Mona.
3. **Filter / sort** — By status or priority? Out of scope for v1; flag as v2.
4. **Dark mode** — Should CSS respect `prefers-color-scheme: dark`? Nice-to-have, at Designer's discretion within `app/styles.css`.
5. **Owner representation** — Plain string vs. `{ name, avatarUrl }`? Recommend plain string for v1 to avoid remote image dependencies.
6. **Launch type** — `node-terminal` (recommended, simplest match for VS Code's `serverReadyAction` recipe) vs. a `type: "node"` variant. Confirm `node-terminal` is acceptable in the learner's VS Code build.
7. **Recent activity formatting** — Free-text string ("Merged PR #42 · 2h ago") vs. ISO timestamp formatted client-side? Recommend free-text for v1.
8. **Summary field** — Mona's brief mentions a "contributor-friendly summary"; should it be required on every project or optional? Recommend optional in schema, always rendered when present.
