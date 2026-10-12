---
name: stack-prototype
description: Build or refine a UI prototype from requirements. Generates portable HTML journeys, reviewable states, and concise companion requirements using a shared UI baseline and component mappings for later framework conversion. Frontend design only — no backend logic. Use when the user wants to design, mock, wireframe, or update a prototype screen or flow.
---

# Prototype Skill

**Scope: frontend design only.** This skill produces UI prototypes and companion requirements. It does not implement backend routes, API handlers, data access, or auth logic.

## Working with stack-spec-workflow

This skill produces product design artifacts: prototypes, concise product behavior specifications, and shared design guidance. `stack-spec-workflow` organizes and tracks delivery of those artifacts; it can also manage their later runtime implementation.

When both skills are active:

- Read the active delivery spec and plan before starting. Keep requirements and the Delivery Plan section together in `docs/specs/`, or the established delivery-workflow location; continue legacy separate plans until consolidation is requested. Keep product specifications in `product-spec/`; do not move delivery documents there or treat a product spec as the delivery spec.
- Execute only the prototype activities covered by the approved delivery phase. The numbered phases below describe this skill's procedure, not additional approval gates. Reuse existing authorization and stop at the delivery plan's phase boundary.
- Update the active delivery plan with completed tasks, validation results, and blockers. Do not create a competing design plan or execution checklist in the product spec. If delivery planning is requested without an active plan, use `stack-spec-workflow` to create the delivery spec and plan.
- Link the delivery spec to its product specs and prototypes. Keep product behavior in the product spec and delivery scope and acceptance in the delivery spec; reference rather than duplicate requirements. Product-flow numbering is independent of delivery IDs.
- Surface conflicts with the delivery spec or referenced approved product requirements before affected work. Follow the delivery workflow's change rules when a requested product change affects locked delivery scope or constraints.
- At an intermediate delivery-phase review, report completed artifacts and deferred steps. Complete the remaining documentation and validation before claiming the entire prototype deliverable is finished.

When used alone, follow this skill's full procedure without creating delivery specs or execution plans unless requested.

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
- Read [the UI baseline and framework mappings](references/ui-baseline.md) before generating or modifying any prototype. Use its neutral sizing and layout conventions even when a future framework is known.
- Read `docs/prototypes/design-system.md` (or an equivalent design system file in the project) if it exists — it is the authoritative record of brand, shared tokens, component patterns, and documented exceptions. Normalize inherited dimensions as described below; an old arbitrary value is not automatically an exception
- Choose the baseline tokens, component roles, sizes, variants, and responsive spans before coding. If no design system exists, use the bundled baseline and record those choices in `design-system.md` after generation
- When editing an existing prototype, identify its flow and canonical shell. Normalize the edited flow and all pages sharing that shell together, including their shared controls and layout tokens. Preserve brand, content, behavior, routes, filenames, and explicit user requirements. Leave unrelated flows untouched; scope any shared design-system updates so their conventions are not silently changed. Record justified exceptions and report every affected page
- Detect any repository-specific instructions (`AGENTS.md`, workspace docs, contribution guides, route conventions) and follow them where they affect naming, output location, or UI terminology

When requirements are incomplete, use this decision order:
1. Explicit user instructions
2. Explicit project requirements and conventions for brand, behavior, and artifact organization (normalize inherited ad hoc dimensions)
3. The shared UI baseline for dimensions and layout, with documented project exceptions
4. Existing prototype patterns for remaining visual decisions

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
- **Responsive** — use the baseline mobile-first layout thresholds when needed; document spans, stacking, and any justified breakpoint exceptions
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

Use [the UI baseline and framework mappings](references/ui-baseline.md) as the single source for dimensional defaults, semantic component classes, and conversion guidance. These are neutral prototype conventions, not the exact defaults of Bootstrap, Material, or shadcn.

Preserve explicit user requirements and project brand choices. Normalize inherited spacing, typography, control sizes, radii, and layouts onto the baseline when editing; record any justified exceptions with their reason and scope. Keep repeated patterns in reusable classes rather than page-specific overrides. Reuse the same inline shared CSS across related pages so the portable files remain consistent.

Before coding, identify each component's role, variant, and size and each layout's container, spans, and responsive behavior. Record these decisions in the authoritative design system during Phase 4. Framework mappings belong in this developer handoff, not in the visible product interface.

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

### Responsive layout

Use the baseline container and optional 12-column grid. Add the 768px and 1024px thresholds only when the content needs a layout change; use the same conditions across the flow. Document column spans and stacking behavior rather than relying on a screenshot to communicate layout intent. CSS custom properties cannot be used directly in media-query conditions, so keep literal query thresholds synchronized with the design-system record.

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

Record material design assumptions and any persistence/reset behavior briefly. Do not add backend endpoints, proposed domain types, or implementation task lists. For requested delivery planning, follow the integration rules above: use a delivery spec in `docs/specs/` with its Delivery Plan section below a horizontal rule, or continue the established legacy files. The product spec remains a design deliverable and contains no execution state.

---

## Phase 4 — Update design system record

Update the authoritative design-system document discovered in Phase 1. If none exists, create `design-system.md` in the prototype output directory. Avoid creating a competing record.

- **If it does not exist:** create it by extracting the complete set of CSS tokens, component roles/classes, size and variant mappings, color palette, responsive spans/breakpoints, exceptions, and JS patterns from the generated prototype(s).
- **If it exists:** compare the new prototype's styles against it. Update normalized tokens, patterns, component mappings, and exceptions, including affected shell pages. Do not leave obsolete dimensional guidance alongside the new baseline.

This file is the authoritative record of shared design conventions. Keep it current so any agent can generate consistent prototypes without reading every HTML file.

`design-system.md` structure:
```markdown
# Design System

Last updated: YYYY-MM-DD

## Canonical Application Shell
(reference page, shared elements, and permitted page differences when applicable)

## CSS Custom Properties
(list all :root variables with values, units where applicable, and semantic meaning; include colors, spacing, sizes, typography, radii, and container width)

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

## Layout and Responsive Breakpoints
(container width/padding, gutters, column spans at each threshold, and stacking behavior)

## Framework Conversion
(component role + prototype class + size/variant; intended Bootstrap, Material UI, and shadcn equivalents for components used; link the baseline mapping reference only if delivered with the project, otherwise record the relevant mappings here)
(native destination defaults take precedence over pixel matching; verify installed versions, theme, and actual local components)

## Exceptions and Normalization
(justified deviations, their reason and scope; normalized flow and shared-shell pages; None when no exceptions exist)

## Vanilla JS Patterns
(standard toggle patterns, outside-click handlers, etc.)
```

---

## Phase 5 — Validate

Read and apply [the prototype review checklist](references/prototype-review.md) before presenting the work. Check portability, requirements, links, assets, and consistency for every output.

When browser automation or preview tooling is available, browser validation is required. Open the generated pages locally or serve the output through a local static server if the browser requires it. Exercise the requested journeys and relevant state URLs at desktop, narrow-mobile, and layout-breakpoint widths. Check console errors and compare the shared shell across pages. Use the available tool's documented commands.

Fix detected issues and rerun affected checks. If tooling is unavailable or a check cannot be completed, report it as unverified with the reason; do not claim validation passed. Capture screenshots when they aid visual review or are requested, and remove only temporary artifacts created for this task.

## Phase 6 — Present and iterate

For a standalone run or the final design-delivery phase, present the completed work after the prototype, companion requirements, design-system updates, and validation are complete. For intermediate delivery phases, use the active delivery plan's review boundary and report remaining work explicitly:

1. Link the launcher or single-screen entry point, the product-spec index, and the relevant flow specifications.
2. Summarize screens, states, fictional data, and any persistence/reset behavior.
3. Note material assumptions and the project instructions or design references followed.
4. Report validation results and any unverified checks or unresolved questions.
5. Invite changes to the existing files.

Apply feedback incrementally unless the user requests a restart. Update affected requirements and design-system guidance, repeat relevant validation, and identify which files changed.
