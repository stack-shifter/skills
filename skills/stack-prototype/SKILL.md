---
name: stack-prototype
description: Build a new UI prototype from requirements. Generates portable HTML journeys, reviewable states, and concise companion requirements. Frontend design only — no backend logic. Use when the user wants to design, mock, or wireframe a screen or flow.
---

# Prototype Skill

**Scope: frontend design only.** This skill produces UI prototypes and companion requirements. It does not implement backend routes, API handlers, data access, or auth logic.

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
/stack-prototype [inline requirements or screen description]
```

Or equivalent plain-language requests such as:
- "Use the stack-prototype skill for this screen"
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

Gather requirements from the user inline or via the questions above. Once you have enough to start, proceed. For minor visual ambiguities, make a reasonable design decision and record it. Ask about missing product behavior that would materially change the journey; do not present an invented rule as an approved requirement.

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

Before generating, outline the requested journey: screens, routes, primary actions, shared navigation, and relevant states. Keep a single-screen request lightweight. Include only states needed to review the flow, such as default, empty, filtered-empty, loading, error, success, or destructive confirmation.

For related screens, designate an existing page or the first generated page as the canonical application shell. Record that choice in the design-system document.

## Phase 2 — Generate prototype

### One or multiple files

A prototype may be **one file or several linked HTML files** depending on the scope:
- Single screen → one `.html` file
- Multi-page flow (e.g. onboarding steps, tabbed workspace, navigation between list and detail) → multiple `.html` files that link to each other with relative `href` values

Keep CSS and JavaScript inline in each HTML file. Link sibling pages with relative URLs such as `<a href="./other-screen.html">`. Bundle referenced images and icons under `assets/` using relative paths; the output directory is the portable unit. Document the files belonging to the flow.

When multiple screens or review scenarios exist, create or update `index.html` as a review launcher. Link to each screen and relevant deterministic scenario, for example `01-records.html?scenario=empty`. Reserve `index.html` for the launcher in a new multi-screen flow; preserve an existing application entry point by placing the launcher in a separate flow directory when needed. A single screen with only its default state needs no launcher.

### Reviewable behavior

- Use fictional fixtures and simulated outcomes without backend calls or authentication. Explain the simulation in the launcher or handoff.
- Make relevant navigation, form validation, cancellation, confirmation, retry, and success actions work. Preserve entered values on recoverable failures and warn before discarding unsaved edits when appropriate.
- Select review states through documented URL parameters so a reviewer can open them directly and reproduce the same starting state. Use a safe default for unknown scenario values.
- Default to deterministic page-local fixtures. If a journey needs browser persistence across screens, document its storage scope and provide a reset action. Do not rely on undocumented browser state to reproduce a scenario.
- Keep shared shell markup, tokens, navigation, controls, and responsive behavior consistent across related pages. Allow intentional differences such as active navigation and content width, and document them.

### File requirements

Each `.html` file must be:
- **Portable** — all CSS in a `<style>` block and all JS inline; bundled relative images/icons are allowed. No external stylesheets, CDN links, remote fonts, imports, or other network dependencies
- **Vanilla JS only** — no frameworks
- **Fully styled with representative dummy data** in every section — no TODOs or placeholders
- **Responsive** — define mobile breakpoints appropriate for the design (common: `640px`, `860px`)
- **Accessible** — prefer semantic HTML, persistent labels, visible keyboard focus, and appropriate accessible names and states. Dialogs and drawers must manage focus and support keyboard dismissal

### File location and naming

Choose the output location using this priority order:

1. User-specified output path
2. Project convention already present in the repo
3. Repository docs or agent instructions that define prototype location
4. A sensible default such as `docs/prototypes/` or the current working directory if no repo convention exists

Use numbered filenames by default, preserving an explicit user instruction or an established project naming scheme when present:

- Assign each new flow the next unused numeric prefix across prototypes and product specs. Start at `01`, use at least two digits, and increment the highest existing flow number by one.
- Give the main screen `<NN>-<slug>.html` and its flow spec `<NN>-<flow-slug>.md`, sharing the same number. Related screens use lowercase letter suffixes in journey order, such as `01a-task-detail.html` and `01b-task-form.html`.
- When extending a flow, reuse its number and assign the next unused letter to new related screens. Preserve existing filenames and numbers; do not renumber files to close gaps. Check for filename collisions before writing.
- Keep `index.html`, `README.md`, `design-system.md`, and asset filenames unnumbered. Scenario query parameters reuse the screen filename rather than receiving separate numbers.

Examples:
- Main screen: `prototypes/01-tasks.html`
- Related screen: `prototypes/01a-task-detail.html`
- Matching flow spec: `product-spec/01-task-management.md`

Always add a comment at the top of each HTML file indicating the target route:
```html
<!-- Route: /your/path -->
```

---

## Design system

The design system for a prototype should come from one of these sources, in priority order:

1. **Explicit user preferences** — apply the requested visual direction
2. **Existing `design-system.md` and project conventions** — preserve the established brand and components
3. **Existing prototype files** — extract patterns when no authoritative design record exists
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

## Phase 3 — Write product specifications

Generate one concise `product-spec/<NN>-<slug>.md` per screen or flow, using a `product-spec/` directory at the project root by default. Follow an explicit user-specified location or an established product-spec convention when present. Keep requirements separate from prototype HTML; do not create a duplicate requirements file alongside the prototype.

Create or update `product-spec/README.md` as a short index with a one-paragraph overview, links to each flow specification, and links to the prototype launcher or individual screens. Preserve existing entries and link central approved requirements rather than repeating them. Even a single-screen project gets an index and one flow spec.

Example layout (prototype location still follows the project convention):

```text
prototypes/
  index.html
  01-records.html
  01a-record-detail.html
  01b-record-form.html
  design-system.md
product-spec/
  README.md
  01-records.md
```

Use relative links from each Markdown file's actual location. For example, a root-level `product-spec/01-records.md` links to `../prototypes/01-records.html`; if prototypes are in `docs/prototypes/`, use `../docs/prototypes/01-records.html` instead. Update links when locations change.

Keep each flow spec short: one or two sentences for the goal, one requirement per bullet, and observable acceptance checks. Link existing approved requirements instead of copying them. Include only relevant states and unresolved questions; write “None” when none remain.

```markdown
# Screen / Flow Name

## Goal
One-sentence purpose of this flow.

## Routes and Prototypes
- `/records` — [Records](../prototypes/01-records.html)
- [Empty state](../prototypes/01-records.html?scenario=empty)
- Requirements source: <link to existing requirements, if present>

## Requirements
- R1: <actor, required behavior, and relevant condition; or a source requirement link>

## States
- Empty: <what the user sees and can do>

## Acceptance Checks
- R1: <observable outcome that demonstrates the requirement>

## Unresolved Questions
- <specific question, or None>
```

Record material design assumptions and any persistence/reset behavior briefly. Do not add backend endpoints, proposed domain types, or implementation task lists. If implementation planning is requested, use separate `docs/specs/plans/<id>_<slug>.plan.md` files and spec references following `stack-spec-workflow`, or the repository's established equivalent. This skill does not require a plan file merely to produce a prototype.

---

## Phase 4 — Update design system record

Update the authoritative design-system document discovered in Phase 1. If none exists, create `design-system.md` in the prototype output directory. Avoid creating a competing record.

- **If it does not exist:** create it by extracting the complete set of CSS tokens, component class names, color palette, breakpoints, and JS patterns from the generated prototype(s).
- **If it exists:** compare the new prototype's styles against it. Update the file if any new tokens, patterns, or component classes were introduced.

This file is the authoritative record of shared design conventions. Keep it current so any agent can generate consistent prototypes without reading every HTML file.

`design-system.md` structure:
```markdown
# Design System

Last updated: YYYY-MM-DD

## Canonical Application Shell
(reference page, shared elements, and permitted page differences when applicable)

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

## Phase 5 — Validate

Read and apply [the prototype review checklist](references/prototype-review.md) before presenting the work. Check portability, requirements, links, assets, and consistency for every output.

When browser automation or preview tooling is available, browser validation is required. Open the generated pages locally or serve the output through a local static server if the browser requires it. Exercise the requested journeys and relevant state URLs at desktop, narrow-mobile, and layout-breakpoint widths. Check console errors and compare the shared shell across pages. Use the available tool's documented commands.

Fix detected issues and rerun affected checks. If tooling is unavailable or a check cannot be completed, report it as unverified with the reason; do not claim validation passed. Capture screenshots when they aid visual review or are requested, and remove only temporary artifacts created for this task.

## Phase 6 — Present and iterate

After the prototype, companion requirements, design-system updates, and validation are complete:

1. Link the launcher or single-screen entry point, the product-spec index, and the relevant flow specifications.
2. Summarize screens, states, fictional data, and any persistence/reset behavior.
3. Note material assumptions and the project instructions or design references followed.
4. Report validation results and any unverified checks or unresolved questions.
5. Invite changes to the existing files.

Apply feedback incrementally unless the user requests a restart. Update affected requirements and design-system guidance, repeat relevant validation, and identify which files changed.
