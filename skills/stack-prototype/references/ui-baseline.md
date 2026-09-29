# UI Baseline and Framework Mappings

Read this reference before generating or modifying a prototype. Choose tokens and component/layout roles before writing HTML; record the actual choices in the delivered `design-system.md`.

This is a neutral prototype baseline, not a claim that frameworks share identical dimensions. Use standalone semantic HTML, inline CSS, and vanilla JavaScript. Framework names below describe the later handoff; do not load their packages, stylesheets, fonts, or scripts into prototypes.

## Tokens

Use these dimensional defaults consistently. Rem equivalents assume the browser's default 16px root size; retain the user's root font preference rather than forcing it to 16px. Preserve project brand colors and choose accessible foreground/background pairs. Declare semantic color tokens such as `--text-primary`, `--text-secondary`, `--brand`, `--on-brand`, `--surface`, `--bg`, and `--border` alongside these dimensions.

```css
:root {
  --space-0: 0;
  --space-xs: 0.25rem; /* 4px */
  --space-sm: 0.5rem;  /* 8px */
  --space-md: 1rem;    /* 16px */
  --space-lg: 1.5rem;  /* 24px */
  --space-xl: 3rem;    /* 48px */
  --control-sm: 2rem;  /* 32px minimum */
  --control-md: 2.5rem; /* 40px minimum */
  --control-lg: 3rem;  /* 48px minimum */
  --text-xs: 0.75rem;  /* 12px: supplementary metadata */
  --text-sm: 0.875rem; /* 14px: labels and secondary text */
  --text-md: 1rem;     /* 16px: body and controls */
  --text-lg: 1.25rem;  /* 20px: subsection headings */
  --text-xl: 1.5rem;   /* 24px: section headings */
  --text-2xl: 2rem;    /* 32px: page title */
  --leading-body: 1.5;
  --leading-heading: 1.25;
  --radius-sm: 0.25rem; /* 4px */
  --radius-md: 0.5rem;  /* 8px: controls */
  --radius-lg: 0.75rem; /* 12px: cards and dialogs */
  --radius-pill: 9999px;
  --container-max: 75rem; /* 1200px */
}
```

Default to 8px gaps within control groups, 16px between fields, 24px card padding/grid gutters, and 48px between major sections. Use parent `gap` where possible; avoid accumulating both child margins and parent gaps. Use 400 body weight and 600 heading/control weight. Badges use 12px text, 4px block/8px inline padding, and the pill radius; keep status readable without relying only on color.

Use tokens for repeated dimensions. Zero, 1px borders, percentages, intrinsic sizing, artwork dimensions, and documented special-purpose sizes are reasonable exceptions. If the baseline cannot serve the content, introduce one clearly named token and document why and where it applies instead of scattering literal values.

## Components

Use semantic elements and reusable classes: `.btn`, `.btn-primary`, `.btn-secondary`, `.btn-quiet`, `.btn-danger`, `.btn-sm`, `.btn-lg`, `.field`, `.field-sm`, `.field-lg`, `.card`, `.badge`, and `.dialog`. Omission of a size modifier means default/medium. Existing names may be retained if their roles and mappings are documented; do not mix framework class vocabularies as though a framework were loaded.

- Primary means the main filled action; secondary means an outlined supporting action; quiet means a low-emphasis text action; danger means a destructive action. Apply disabled, loading, hover, and visible focus states consistently.
- Align mixed-size controls centrally in action rows so flex stretching does not erase their size differences. Match single-line input/select and button minimum heights by size. Use 14px text for small controls and 16px for default/large controls. Use 8/16/24px inline padding for small/default/large buttons, with 4px block padding; fields use 8px inline and 4px block padding. Allow content-driven growth. Do not force multiline inputs into single-line heights.
- Icon-only controls use square dimensions matching their size role and an accessible name. Ensure touch use remains comfortable: on coarse pointers, increase small/default interactive targets to at least the 48px control token and allow control groups to wrap. Do not shrink a target merely to match an icon.
- Cards and dialogs use 24px interior padding and the 12px radius. Use semantic headings, labeled fields, help/error text, and explicit action groups so these structures map naturally to framework components.
- Use real buttons for actions and anchors for navigation. Preserve keyboard behavior and focus handling, including modal focus containment, Escape dismissal, and focus restoration. CSS classes alone do not implement accessible interactions.

Example sizing rules (add the project's colors, borders, variants, and states):

```css
*, *::before, *::after { box-sizing: border-box; }
body {
  margin: 0;
  font-family: system-ui, sans-serif;
  font-size: var(--text-md);
  line-height: var(--leading-body);
}
.btn, .field {
  --control-size: var(--control-md);
  min-block-size: var(--control-size);
  font: inherit;
  border: 1px solid var(--border, currentColor);
  border-radius: var(--radius-md);
  padding: var(--space-xs) var(--space-md);
}
.btn { font-weight: 600; white-space: normal; }
.btn-sm, .field-sm { --control-size: var(--control-sm); font-size: var(--text-sm); }
.btn-lg, .field-lg { --control-size: var(--control-lg); }
.btn-sm { padding-inline: var(--space-sm); }
.btn-lg { padding-inline: var(--space-lg); }
.field { padding-inline: var(--space-sm); max-inline-size: 100%; }
.btn-icon {
  inline-size: var(--control-size);
  block-size: var(--control-size);
  padding: var(--space-0);
  display: inline-grid;
  place-items: center;
  flex-shrink: 0;
}
.actions { display: flex; flex-wrap: wrap; align-items: center; gap: var(--space-sm); }
@media (pointer: coarse) {
  .btn, .field { --control-size: var(--control-lg); }
}
```

## Containers and grids

Start stacked. Use a centered container capped at 1200px including its padding, with 16px inline padding on mobile and 24px from 768px. Add a 12-column grid only where useful; flex/stack layouts are appropriate for simple action groups and single-column content.

Use 768px and 1024px minimum-width thresholds only when layout changes are needed. Media queries require literal thresholds rather than CSS custom properties. Record their numeric values in `design-system.md`; do not invent a separate breakpoint per page. Explicitly document exceptions required by the content or user.

```css
.container {
  inline-size: 100%;
  max-inline-size: var(--container-max);
  margin-inline: auto;
  padding-inline: var(--space-md);
}
.grid { display: grid; grid-template-columns: repeat(12, minmax(0, 1fr)); gap: var(--space-lg); }
.grid > * { min-inline-size: 0; grid-column: span 12; }
@media (min-width: 768px) {
  .container { padding-inline: var(--space-lg); }
  .grid > .summary-card { grid-column: span 6; }
}
@media (min-width: 1024px) {
  .grid > .summary-card { grid-column: span 4; }
}
```

For this example, record `summary-card: 12 columns initially, 6 from 768px, 4 from 1024px; 24px gutter`. Use explicit spans for main/sidebar layouts as well. Avoid absolute positioning for normal page layout and unexplained fixed column widths. Allow tables to scroll within a bounded region where appropriate without creating page overflow.

## Conversion handoff

Record the roles, prototype classes, sizes/variants, grid spans, and relevant destination equivalents in the project's design-system document. Document components actually used; do not copy a catalog of unused components or reference a skill-local file that the recipient cannot access.

When conversion is requested, inspect the destination's installed library versions, theme, and local component implementations first. Use native components and default sizing, preserving hierarchy, action emphasis, behavior, and responsive intent. Small visual changes are expected. Do not add pervasive height, padding, radius, or font overrides just to reproduce prototype pixels. Honor explicit brand or dimensional requirements through the destination's established theme conventions and document necessary exceptions.

### Spacing mapping

These examples assume unmodified framework spacing scales and a 16px root. Names/numbers are not interchangeable across libraries. Check the project's theme before using them.

| Prototype token | Nominal value | Bootstrap spacing suffix | Material UI `theme.spacing(n)` | Tailwind spacing suffix |
| --- | --- | --- | --- | --- |
| `--space-0` | 0 | `0` | `0` | `0` |
| `--space-xs` | 4px | `1` | `0.5` | `1` |
| `--space-sm` | 8px | `2` | `1` | `2` |
| `--space-md` | 16px | `3` | `2` | `4` |
| `--space-lg` | 24px | `4` | `3` | `6` |
| `--space-xl` | 48px | `5` | `6` | `12` |

For example, 24px padding maps to Bootstrap `p-4`, Material UI `sx={{ p: 3 }}`, or Tailwind `p-6`. Material UI custom spacing functions/arrays may not accept fractional factors; use the project's supported spacing mechanism. shadcn layouts use the destination project's Tailwind utilities rather than a separate shadcn grid API.

### Component and layout mapping

Material UI is the concrete Material implementation illustrated here, not a universal API for every Material library. These are semantic starting points; inspect the actual destination API, including shadcn's local source and primitive choice.

| Prototype role | Bootstrap | Material UI | shadcn / Tailwind |
| --- | --- | --- | --- |
| Primary action | `.btn.btn-primary` | `Button variant="contained"` | `Button variant="default"` |
| Outlined supporting action | `.btn.btn-outline-secondary` | `Button variant="outlined"` | `Button variant="outline"` |
| Quiet action | Low-emphasis button per project convention; keep action semantics | `Button variant="text"` | `Button variant="ghost"` |
| Destructive action | `.btn.btn-danger` | `Button color="error"` with appropriate variant | `Button variant="destructive"` |
| Small/default/large button | `.btn-sm` / no size modifier / `.btn-lg` | `size="small"` / `"medium"` / `"large"` | `size="sm"` / `"default"` / `"lg"` |
| Icon action | Button with accessible name and icon | `IconButton` | `Button` with supported icon size and accessible name |
| Text field | Labeled `.form-control`; `.form-control-sm` / default / `.form-control-lg` | `TextField` with `size="small"` or `"medium"`; no native large size | Labeled `Input`; no assumed small/large size prop |
| Select | Labeled `.form-select` and supported size modifier | `Select` / `TextField select` with labeling | `NativeSelect` or `Select` appropriate to behavior; verify local API |
| Card | `.card` with `.card-body` | `Card` / `CardContent` / `CardActions` | `Card` / `CardHeader` / `CardContent` / `CardFooter` |
| Status badge | `.badge` and semantic color | `Chip` for status labels; `Badge` for counts/indicators | `Badge` and appropriate variant |
| Dialog | Modal with title/body/footer | `Dialog` / `DialogTitle` / `DialogContent` / `DialogActions` | `Dialog` and labeled content; `AlertDialog` for alert confirmations |
| Content container | `.container` / project container convention | `Container` with native max-width policy | Project container with Tailwind width/padding utilities |
| Responsive grid | `.row`, gutters, responsive `.col-*` spans | `Grid` container/items with responsive sizes | `grid grid-cols-12`, `gap-*`, responsive `col-span-*` |
| Action/field stacks | Flex or stack helpers and gap utilities | `Stack` with spacing | Flex/grid and gap utilities |

For inputs without all three size equivalents, use the closest native size that preserves emphasis and usability; document the mapping rather than inventing props. Map prototype typography and radii to existing theme roles instead of forcing the numerical prototype values.

Keep the sequence of responsive layouts (for example 12 → 6 → 4 spans), using the destination's appropriate native thresholds. Bootstrap, Material UI, and Tailwind breakpoint names and widths differ. Verify behavior around the destination thresholds; do not blindly copy a `md` label or assume that a 1024px prototype threshold is every framework's desktop breakpoint. If exact threshold behavior is an explicit requirement, document a deliberate theme breakpoint exception.

### Official references

Consult the installed version's documentation and theme, using these starting points:

- Bootstrap: [spacing](https://getbootstrap.com/docs/5.3/utilities/spacing/), [buttons](https://getbootstrap.com/docs/5.3/components/buttons/), [forms](https://getbootstrap.com/docs/5.3/forms/form-control/), [grid](https://getbootstrap.com/docs/5.3/layout/grid/), [breakpoints](https://getbootstrap.com/docs/5.3/layout/breakpoints/), [components](https://getbootstrap.com/docs/5.3/components/card/).
- Material UI: [spacing](https://mui.com/material-ui/customization/spacing/), [buttons](https://mui.com/material-ui/react-button/), [text fields](https://mui.com/material-ui/react-text-field/), [grid](https://mui.com/material-ui/react-grid/), [breakpoints](https://mui.com/material-ui/customization/breakpoints/), [dialogs](https://mui.com/material-ui/react-dialog/).
- shadcn: [components](https://ui.shadcn.com/docs/components), [buttons](https://ui.shadcn.com/docs/components/button), [inputs](https://ui.shadcn.com/docs/components/input), [theming](https://ui.shadcn.com/docs/theming). Tailwind: [spacing utilities](https://tailwindcss.com/docs/padding), [grid spans](https://tailwindcss.com/docs/grid-column), [responsive design](https://tailwindcss.com/docs/responsive-design).
