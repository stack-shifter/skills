# Prototype Review Checklist

Apply relevant checks to the requested journey. Mark each as passed, failed, not applicable, or unverified with a short reason. Fix failures before delivery; report checks that tooling cannot verify.

## Portable output and handoff

- Unless an explicit instruction or established naming convention overrides it, prototype screens and flow specs share numeric prefixes; related screens use letter suffixes. Existing numbers remain stable, filenames do not collide, and index/design-system/asset files remain unnumbered.

- CSS and JavaScript are inline; images and icons resolve from bundled relative asset paths. No remote fonts, libraries, data requests, or hidden local dependencies are required.
- Relative page links work from the delivered directory. The launcher exists when multiple screens or review scenarios exist and preserves existing entry points.
- Fictional data and simulated outcomes require no authentication or backend.
- The product-spec directory contains a concise README index and one spec per screen or flow. Index links and cross-directory prototype/scenario links resolve from their actual locations; no duplicate requirements file is generated alongside the HTML.
- Flow specifications identify routes, prototype links, relevant states, observable acceptance checks, and unresolved questions without implementation task lists or duplicated source requirements.
- The authoritative design-system record reflects the delivered tokens, components, and canonical shell. Explicit user preferences take precedence over inherited defaults.

## Journeys and state

- Open each launcher destination and scenario URL directly, then reload it to confirm a reproducible starting state. Unknown scenario values fall back safely.
- Follow primary actions, back links, and navigation between related screens. Confirm selected navigation matches the destination.
- Exercise applicable validation, submission, success, failure/retry, cancellation, and destructive confirmation paths. Recoverable errors retain entered data; cancellation preserves or discards it as documented.
- If browser persistence is used, verify its documented scope, a working reset action, and predictable behavior after reset. Direct review scenarios must not depend on a previous reviewer's actions.

## Responsive layout and shared design

- Inspect representative desktop and narrow-mobile widths and each layout-changing breakpoint, including immediately above and below the boundary.
- Check for unintended page overflow, clipped text, unreachable actions, and unusable tables, menus, drawers, or dialogs.
- Compare the canonical shell across related screens at the same widths: header, navigation, control sizes, typography, colors, spacing, and footer behavior.
- Confirm only documented differences such as active navigation or content widths vary. Check for page-specific CSS overriding shared shell rules.

## UI baseline and conversion handoff

- Read the bundled UI baseline before generation. Verify spacing, typography, radii, control sizes, and container widths use its tokens rather than unexplained one-off values. Explicit user requirements and justified exceptions are recorded with their scope.
- Buttons and single-line fields use named small/default/large roles and consistent minimum heights. Icon controls have matching square dimensions, accessible names, and usable touch targets. Long labels and enlarged text can grow without clipping.
- Repeated components use semantic classes with clear roles, variants, and sizes; related pages share the same rules. Borders, artwork, and documented special-purpose dimensions need not use spacing tokens.
- The design-system record includes token values/units, component mappings, container padding, grid gutters/spans, and stacking behavior at each used threshold. Literal CSS media queries match the recorded thresholds.
- Conversion guidance maps components actually used to Bootstrap, Material UI, and shadcn/Tailwind, flags missing size equivalents, and favors native destination defaults over pixel matching. It does not imply equal breakpoint names mean equal widths or invent unsupported component props.
- When editing, normalize the whole affected flow and pages sharing its canonical shell. Preserve brand, behavior, routes, filenames, and explicit user requirements; verify unrelated flows remain unchanged. Report normalized pages and justified exceptions.

## Accessibility and runtime

- Navigate with the keyboard: verify logical order, visible focus, semantic controls, persistent form labels, and accessible names and states.
- Open and dismiss dialogs/drawers; verify Escape behavior, focus containment for modal surfaces, and focus restoration to the trigger.
- Check status and error messaging, contrast, non-color-only cues, usable touch targets, and reduced-motion behavior where animation exists.
- Inspect console errors and failed asset loads on each screen and after relevant interactions.

Browser checks require available browser or preview tooling. Static source inspection alone does not establish rendering, keyboard, or interaction correctness. Record unavailable checks as unverified and summarize them in the final handoff.
