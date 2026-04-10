---
name: spec-workflow
description: Drives a spec-first workflow where the spec is the locked source of truth and implementation runs through plan mode. Use whenever the user wants a feature spec drafted with explicit clarification questions, locked requirements in `docs/specs`, or a "do the next chunk then check in" implementation loop.
---

# Spec Workflow Skill

## Purpose

Keep one stable spec per feature as the source of truth. Get user approval before starting implementation. Check in after each logical chunk of work before continuing.

- **Spec = what** — locked requirements, stable across the entire feature
- **Plan mode = how** — in-session design derived from the spec before each implementation run

## When to Use

- Spec-first workflow for a new feature
- Stable requirements document driving phased implementation
- "Do only the next phase" loop
- Locked requirements with explicit user approval gates

Typical trigger phrases:

- `Spec: build ...`
- "create a spec for this feature"
- "implement the next phase"
- "use the spec as locked requirements"
- "set this up in docs/specs"

Do not use for trivial one-off edits unless the user explicitly asks for the workflow.

## Canonical Layout

```text
docs/
  specs/
    010_task-management.spec.md
    020_user-invitations.spec.md
```

If `CLAUDE.md` exists in the repository, treat it as an additional instruction source alongside `AGENTS.md`.

## Naming

File names: `<id>_<slug>.spec.md`

Examples: `010_task-management`, `011_user-invitations`, `012_billing-webhooks`

To choose the next prefix, list existing spec files and increment by 1. Do not use vague slugs like `stuff`, `misc`, or `new-feature`.

## Spec File

`docs/specs/<id>_<slug>.spec.md` must contain:

```md
# <Feature Title>

## Overview

## Scope / Non-Scope

### In Scope

### Out of Scope

## Requirements

## Acceptance Criteria

## What This Touches

> A loose list of areas, files, and surfaces likely affected. For human reference — not an ordered plan.

- ...
```

When drafting, mark every assumption or gap inline rather than guessing:

```
[NEEDS CLARIFICATION: <specific question>]
```

Examples:

```
- Users receive an email on assignment [NEEDS CLARIFICATION: should this also fire for reassignment?]
- Auth method not specified [NEEDS CLARIFICATION: email/password, SSO, or OAuth?]
```

Do not resolve ambiguities by assuming. Surface them in the draft and ask the user to fill them in before the spec is treated as locked.

After writing a new spec, always output a summary card:

```
Spec:    <id>_<slug>.spec.md
Branch:  dev-<slug>
Summary: <one-sentence description of the feature>
Open Questions: <count of [NEEDS CLARIFICATION] markers>
```

## Holistic Validation Pass

Run this before drafting or executing against a spec:

1. **Surface audit** — find every symbol, config key, route, UI string, contract, and behavior the spec touches. Identify all impacted layers — app, UI, API, data, infrastructure, tests, docs — not just files already named.

2. **Scope consistency** — check for contradictions between Overview, Scope, Requirements, and Acceptance Criteria. If an acceptance criterion requires changing an out-of-scope layer, narrow the criterion or bring the layer into scope.

3. **Phase executability** — verify each planned phase can be validated using only work from that phase and earlier. Reorder phases when later-phase work is a prerequisite.

4. **Explicit ownership** — name every caller, consumer, adapter, handler, test, doc, and config that must change. Do not leave high-risk coupling as implied cleanup.

5. **Contract boundaries** — distinguish product, application, storage, integration, and infrastructure behavior. State clearly when a change affects one boundary but intentionally leaves another unchanged.

Present findings in order: contradictions → missing surface → phase/ordering risks → test gaps → doc fixes.

## Workflow

### Stage 1: Create or refine the spec

1. Gather context from the repo and the user request.
2. Run the holistic validation pass.
3. Draft the spec in `docs/specs/` (or the repo's established location). Mark every ambiguity or missing detail with `[NEEDS CLARIFICATION: <specific question>]` rather than assuming.
4. Output the summary card.
5. After outputting the summary card, print all `[NEEDS CLARIFICATION]` markers as a numbered list so the user can answer them without opening the file. The spec is not locked while markers remain.

When a spec already exists, treat it as the source of truth and run the holistic validation pass before accepting it as execution-ready.

### Stage 2: Implementation

Before starting implementation, check the current git branch. If on `main` or `master`, ask the user to confirm before creating a `dev-<slug>` branch (e.g., `dev-010-task-management`). Do not create it automatically. If already on a `dev-*` branch, proceed.

Use the locked spec as requirements. Get user approval before executing each phase of work. Stop after each phase and wait for explicit instruction before continuing.

## Spec Lock Rule

Treat the spec as locked once the user approves it or asks for implementation without requesting requirement changes.

Do not start implementation while any `[NEEDS CLARIFICATION]` markers remain. Resolve them first, then treat the cleaned spec as the lockable source of truth.

During implementation, the user may change direction. Use judgment to decide whether the change is small or large:

**Small change** — absorb it. Update the spec inline and continue. A change is small if it:
- doesn't invalidate work already completed
- doesn't change the core approach or architecture
- can be absorbed into the current or next phase without restructuring

**Large change** — stop and escalate before touching anything. A change is large if it:
- invalidates completed work
- significantly expands or restructures scope
- changes the core approach or introduces new surfaces

Do not silently absorb a large change by treating it as a small one.

## Spec Change Escalation

When a change is too large to absorb inline, say so explicitly before proceeding. Present:

1. What the user is asking for
2. Why it is a large change (what it invalidates or restructures)
3. Proposed spec revision
4. Revised implementation approach

Wait for user approval before editing the spec or continuing implementation.

## Choosing the Active Spec

When multiple specs exist, determine the active file in this order:

1. Explicit user-provided file or feature ID
2. Files referenced in the current task context
3. Files matching the current branch or active feature name
4. The only matching spec in the repository

If multiple candidates remain, ask the user which to use.

## Guardrails

- Do not advance past the current phase without explicit user instruction.
- Do not silently change locked requirements by editing the spec during implementation.
- Do not guess which spec is active when multiple candidates exist.
- Do not store execution state in the spec.
- Use the spec as the single source of truth.
