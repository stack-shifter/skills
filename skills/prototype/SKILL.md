---
name: prototype
description: Build a new UI prototype from requirements. Generates self-contained HTML file(s) plus a product spec doc. Frontend design only — no backend logic. Use when the user wants to design, mock, or wireframe a screen or flow.
---

# Prototype Skill

**Scope: frontend design only.** This skill produces UI prototypes and implementation specs. It does not implement backend routes, API handlers, data access, or auth logic.

## Agent compatibility

This workflow is intended to be reusable across agent runtimes, including Claude, Codex, Gemini, or any other agent that can read instructions and create files.

Agent-specific wrappers may differ, but the workflow should stay the same:
- Claude-style invocation may use a slash command
- Codex-style invocation may be triggered by naming the skill or asking for a prototype explicitly
- Other agents may use the phase instructions directly as a prompt or embedded skill

If the host agent does not support native skills, treat this file as an execution playbook and follow the phases in order.

## How to invoke

Possible invocation styles:

```
/prototype [inline requirements or screen description]
```

Or equivalent plain-language requests such as:
- "Use the prototype skill for this screen"
- "Generate a prototype for this flow"
- "Create HTML wireframes and a companion spec from these requirements"

If no requirements are given, ask:
1. What screen or flow is this? (name + route if applicable)
2. What are the key entities or data shown?
3. What are the primary user actions?
4. Are there any states to show? (empty, loading, error, confirmation)
5. Does this span multiple pages that should link to each other?

---

## Phase 1 — Requirements

Gather requirements from the user inline or via the questions above. Once you have enough to start, proceed. For ambiguous details, make a reasonable design decision and note it in the post-generation summary.

Before writing any code:
- Look for existing prototypes in the project to use as structural and visual reference
- Read `docs/prototypes/design-system.md` (or an equivalent design system file in the project) if it exists — it is the authoritative record of shared CSS tokens and component patterns for this project
- If no existing design system or prototypes are found, establish sensible defaults and document them when creating `design-system.md` after generation
- Detect any repository-specific instructions (`AGENTS.md`, workspace docs, contribution guides, route conventions) and follow them where they affect naming, output location, or UI terminology

When requirements are incomplete, use this decision order:
1. Explicit user instructions
2. Project-level conventions already present in the repo
3. Existing prototype patterns in the repo
4. Sensible defaults documented in the summary and companion spec

---

## Phase 2 — Generate prototype

### One or multiple files

A prototype may be **one file or several linked HTML files** depending on the scope:
- Single screen → one `.html` file
- Multi-page flow (e.g. onboarding steps, tabbed workspace, navigation between list and detail) → multiple `.html` files that link to each other with relative `href` values

Each HTML file is self-contained (all CSS and JS inline) but may reference sibling files via `<a href="./other-screen.html">`. Clearly document which files belong to the same prototype in the companion spec doc.

### File requirements

Each `.html` file must be:
- **Self-contained** — all CSS in a `<style>` block, all JS inline — no external stylesheets, no CDN links, no imports
- **Vanilla JS only** — no frameworks
- **Fully styled with representative dummy data** in every section — no TODOs or placeholders
- **Responsive** — define mobile breakpoints appropriate for the design (common: `640px`, `860px`)
- **Accessible** — use `role` and `aria-label` where appropriate

### File location and naming

Choose the output location using this priority order:

1. User-specified output path
2. Project convention already present in the repo
3. Repository docs or agent instructions that define prototype location
4. A sensible default such as `docs/prototypes/` or the current working directory if no repo convention exists

If the project uses numbered prototype files, follow the existing numbering scheme. If no numbering convention exists, use clear descriptive slugs without inventing arbitrary numbers.

Examples:
- Repo with prototype convention: `docs/prototypes/<NN>-<slug>.html`
- Generic repo without numbering: `prototypes/<slug>.html`
- No existing convention: `<current-working-directory>/<slug>.html`

Always add a comment at the top of each HTML file indicating the target route:
```html
<!-- Route: /your/path -->
```

---

## Design system

The design system for a prototype should come from one of these sources, in priority order:

1. **Existing `design-system.md`** in the project — read and follow it exactly
2. **Existing prototype files** in the project — extract patterns from them
3. **User-specified preferences** — if the user describes a visual style, apply it
4. **Sensible generic defaults** — when nothing exists, use the pattern below as a starting point and adapt it to fit the project's context

### Generic defaults (adapt to fit the project)

These are starting-point conventions. Adjust colors, fonts, and component shapes to match the product's brand and tone.

**CSS custom properties** — always declare in `:root`:
```css
:root {
  --text-primary: #111827;
  --text-secondary: #6b7280;
  --border: #e5e7eb;
  --brand: #2563eb;        /* adjust to project brand color */
  --surface: #ffffff;
  --bg: #f9fafb;
}
```

**Body:** `background: var(--bg); font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; color: var(--text-primary);`

**Status indicators** — adapt label names and colors to the domain:
```css
.pill-active   { background: #d1fae5; color: #065f46; }
.pill-pending  { background: #fef9c3; color: #92400e; }
.pill-review   { background: #ede9fe; color: #5b21b6; }
.pill-inactive { background: #f3f4f6; color: #6b7280; }
/* pill base: font-size:11px; font-weight:700; padding:3px 10px; border-radius:20px; text-transform:uppercase; */
```

**Typography scale:**
| Use | Size | Weight |
|-----|------|--------|
| Page title | `22–24px` | `800` |
| Section heading | `15–16px` | `700` |
| Body / row text | `13–14px` | `400–500` |
| Label / meta | `11–12px` | `400–600` |

**Spacing:** multiples of 4px — common values: `4 8 12 16 20 24 28 32px`.

**Border radius:** cards/modals `10–12px`, buttons `8px`, pills `20px`, inputs `8px`.

**Buttons:**
```css
.btn     { padding: 8px 16px; font-size: 13px; font-weight: 600; border-radius: 8px; cursor: pointer; }
.btn-sm  { padding: 6px 12px; font-size: 12px; }
.btn-xs  { padding: 4px 10px; font-size: 11px; }
.btn-primary   { background: var(--brand); color: #fff; border: none; }
.btn-secondary { background: var(--surface); color: var(--text-primary); border: 1px solid var(--border); }
```

**Cards:** `background: var(--surface); border: 1px solid var(--border); border-radius: 10px;`

**Modals:** fixed overlay `rgba(0,0,0,0.5)`, centered dialog `max-width: 760px`, scrollable body, close button top-right.

### Vanilla JS interactivity

Toggle visibility with `.open` / `.active` classes. Standard pattern:
```js
document.querySelector('.trigger').addEventListener('click', (e) => {
  e.stopPropagation();
  document.querySelector('.target').classList.toggle('open');
});
document.addEventListener('click', () => {
  document.querySelector('.target')?.classList.remove('open');
});
```

### Responsive breakpoints

Define breakpoints appropriate to the design. When using the generic defaults:
```css
@media (max-width: 860px) {
  /* collapse secondary columns */
}
@media (max-width: 640px) {
  /* stack layout, simplify nav, convert tables to cards */
}
```

---

## Phase 3 — Iterate

After generating:
1. Summarize: what files were created, what screens/states are included, what dummy data is used
2. Note design decisions made for ambiguous requirements
3. Note any project conventions that were inferred rather than explicitly provided
4. Invite changes: "Tell me what to adjust — I'll update the same file(s)"

Never start over unless the user explicitly asks. Apply changes incrementally. When multiple files exist, be explicit about which file is being changed.

---

## Phase 4 — Generate product spec doc

Always generate a companion **Markdown spec file** alongside the prototype. Place it in the same directory as the HTML file(s):

`<output-dir>/<slug>.md`

Use this structure:

```markdown
# Screen / Flow Name

Primary wireframe(s): `<slug>.html` (list all files if multiple)

## Goal
One-sentence description of what this screen or flow demonstrates or enables.

## Routes
- `/path/to/screen`
- Query parameters (if any): `?filter=value`

## Frontend scope
- List of UI components and interactions to implement
- Note which states are shown (empty, error, loading, etc.)

## Interaction model
Rules and constraints for how the UI should behave. Be specific — avoid vague instructions.

## Shared domain types to introduce
- `TypeName` — brief description

## Backend scope (after frontend)
- API endpoints needed to power this screen
- Keep brief — frontend is built first

## Recommended implementation slices
1. First slice (shell + layout)
2. Second slice (data + interactions)
3. ...

Agent guidance: rules, watch-outs, or constraints an implementing agent should follow.
```

---

## Phase 5 — Update design system record

After generating, check whether a `design-system.md` file exists in the prototype output directory.

- **If it does not exist:** create it by extracting the complete set of CSS tokens, component class names, color palette, breakpoints, and JS patterns from the generated prototype(s).
- **If it exists:** compare the new prototype's styles against it. Update the file if any new tokens, patterns, or component classes were introduced.

This file is the authoritative record of shared design conventions. Keep it current so any agent can generate consistent prototypes without reading every HTML file.

`design-system.md` structure:
```markdown
# Design System

Last updated: YYYY-MM-DD

## CSS Custom Properties
(list all :root variables with hex values and their semantic meaning)

## Color Palette
(document all colors used, named by semantic role)

## Typography Scale
(size + weight for each text role)

## Spacing
(scale used, common padding/margin values)

## Border Radius
(values by context)

## Component Classes
### Buttons
### Cards
### Status Pills / Badges
### Navigation
### Modals
### Tabs
### Tables / Lists
(add sections as new components are introduced)

## Responsive Breakpoints
(breakpoint values and what changes at each)

## Vanilla JS Patterns
(standard toggle patterns, outside-click handlers, etc.)
```

---

## Phase 6 — Portability and quality check

Before finishing, validate that the generated output is reusable by other agents and understandable by a developer who has not read this skill file.

Check all of the following:
- No instructions depend on one specific agent runtime, model, or slash-command syntax
- Output paths follow project conventions when available, otherwise use explicit documented defaults
- HTML files are fully self-contained and do not rely on external frameworks or hidden local assets
- The companion spec explains enough context that another agent can continue the work without rereading the whole conversation
- Any assumptions about routes, entities, or visual style are called out explicitly
- If the repo has agent instructions or product docs, the summary names which ones were followed

If any of these checks fail, fix the prototype or spec before presenting the result.

## Phase 7 — Validate in browser (optional)

If the host environment has a browser automation or preview tool available, use it to validate the generated HTML before finishing.

Preferred checks:
1. Open each generated HTML file locally in a browser or preview tool
2. Confirm the layout renders without broken styles
3. Check for JavaScript console errors
4. Verify responsive behavior at desktop and mobile widths
5. Capture a screenshot only if the user asked for one or if it helps document the result

If `playwright-cli` is available, one valid approach is:
```bash
which playwright-cli 2>/dev/null || npx playwright-cli --version 2>/dev/null
```

Then, for each file:
1. `playwright-cli open file:///absolute/path/to/file.html`
2. `playwright-cli console`
3. Resize to representative desktop and mobile widths
4. Optionally take a screenshot
5. `playwright-cli close`

Clean up any temporary screenshots or automation artifacts created during validation.

If no browser automation or preview tool is available, skip this phase and state that validation was not run.
