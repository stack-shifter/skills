---
name: spec-workflow
description: Drives a spec-first workflow where the spec is the locked source of truth and mutable plan files track phased implementation. Use whenever the user wants a feature spec drafted with explicit clarification questions, locked requirements in `docs/specs`, a persistent phased implementation plan, or a "do the next chunk then check in" implementation loop.
---

# Spec Workflow Skill

## Purpose

Keep one stable spec per feature as the source of truth. Get user approval before starting implementation. Check in after each logical chunk of work before continuing.

- **Spec = what** — locked requirements, stable across the entire feature
- **Plan = how** — persistent phases, tasks, validation results, and blockers derived from the spec

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

Prefer the repository's established spec-and-plan convention. Otherwise, use:

```text
docs/
  specs/
    010_task-management.spec.md
    plans/
      010_task-management.plan.md
```

If `CLAUDE.md` exists in the repository, treat it as an additional instruction source alongside `AGENTS.md`.

## Naming

File names: `<id>_<slug>.spec.md` and `<id>_<slug>.plan.md`. Use the same numeric prefix and stable slug for the feature across both files.

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

- ...

## Plan References

- `docs/specs/plans/<id>_<slug>.plan.md`
```

### Keep specs concise

Write the spec as a readable requirements document. Scale its length to the feature; aim for about one page for a small feature, without cutting necessary behavior or constraints to meet a word limit.

- Keep the Overview to one or two sentences explaining the feature and its purpose.
- Use short scope bullets to define boundaries. Do not repeat the full requirements under In Scope.
- Write one requirement per bullet with a stable ID such as `R1`. State who or what acts, the required behavior, and any condition that changes the outcome. Prefer plain language and concrete verbs over vague terms such as “robust” or “seamless.”
- Include error behavior, permissions, limits, and contracts only when they affect correctness or acceptance. Keep technical constraints that are actual requirements; move implementation choices, task breakdowns, file inventories, and validation commands to the plan.
- Make acceptance criteria observable checks tied to requirement IDs. Use them to show how success is recognized, rather than repeating each requirement verbatim.
- Keep What This Touches to a few affected areas or surfaces. Put the detailed ownership and file list from the holistic audit in the plan.
- State each rule once. Link to existing contracts or supporting references instead of copying long explanations. Omit background essays, speculative future work, and sections beyond the template unless needed to resolve a concrete ambiguity.

Example requirement and acceptance check:

```md
## Requirements

- R1: Only the task owner can delete a task.

## Acceptance Criteria

- R1: An owner can delete their task; another user's attempt is denied and leaves the task unchanged.
```

Before presenting the spec, remove repetition and implementation detail, then confirm that the remaining requirements are clear and testable. Concision must not hide unresolved questions or omit necessary constraints.

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
Plan:    docs/specs/plans/<id>_<slug>.plan.md (planned or created)
Branch:  dev-<slug>
Summary: <one-sentence description of the feature>
Open Questions: <count of [NEEDS CLARIFICATION] markers>
```

## Plan File

Use one plan file by default. Use multiple plans only for distinct execution tracks, such as backend and frontend work or a separate migration. Reference every plan from the spec, give each a distinct scope, and operate on one active plan per run unless the user explicitly asks otherwise. Record dependencies between plans in their tasks and gates.

Each plan should use this structure, repeating the phase section as needed:

```md
# <Feature Title> Plan

Spec: `docs/specs/<id>_<slug>.spec.md`

## Phase 1 - <Outcome>

### Tasks

- [ ] `P1.T1` <small, reviewable task>
- [ ] `P1.T2` <small, reviewable task>

### Expected Files

- `path/to/file.ts`

### Validation

- <command or manual check>: not run

### Gate

- <observable completion condition>: not verified

## Blockers

- None currently.
```

Keep the plan execution-oriented. Refine ambiguous or oversized tasks before coding: clarify wording, split tasks, and update file lists, checks, or gates. Preserve existing task IDs when possible and assign new IDs to added tasks. Record blockers and concise validation evidence as work progresses.

If code has drifted from the plan but still matches the spec, update the plan to reflect reality before continuing. Do not hide requirement changes inside plan refinements. Reconcile affected plans whenever an authorized spec revision occurs.

## Holistic Validation Pass

Run this before drafting, revising, or executing a spec or plan:

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

### Stage 2: Generate the phased plan

Once the spec is locked, create or revise its referenced plan file. Keep execution state in the plan so work can resume across sessions. Read the spec and current code before defining phases.

Break the work into ordered, reviewable phases. For each phase, define:

- the outcome and spec requirements it satisfies
- atomic tasks with stable IDs such as `P1.T1`, checkboxes, and expected files or surfaces to change
- validation commands or manual checks appropriate to the change
- a gate: observable conditions that must pass before the phase is complete

Run the phase executability check against this plan. Each phase must be verifiable using its own work and earlier phases. Include documentation updates when behavior or workflows change.

Present the phase outline and identify the next phase for user approval. Treat an explicit instruction to execute a presented phase as approval; do not ask again for the same work.

When resuming, read the spec and active plan and compare them with the current code. Reconcile completed work and remaining tasks in the plan without changing requirements. Select the earliest phase whose tasks, validation, or gate remain incomplete, even if all its task checkboxes are checked. If the plan is missing, generate it before implementing.

### Stage 3: Implement and review one phase

Before starting implementation, check the current git branch. If on `main` or `master`, ask the user to confirm before creating a `dev-<slug>` branch (e.g., `dev-010-task-management`). Do not create it automatically. If already on a `dev-*` branch, proceed.

Use the locked spec as requirements and execute the approved phase:

1. Name the spec, active plan, phase, and task IDs. Implement unchecked tasks in order, keeping changes scoped to the phase. Mark each task `[x]` in the plan immediately after its implementation work is complete.
2. Run the planned validation and required repository checks. Broaden testing when shared behavior or regression risk warrants it; a full suite is not required for every phase unless repository instructions require it.
3. Record validation commands and results and the gate status in the plan. A phase is complete only when its tasks are done, validation passes, and the gate is satisfied.
4. Report completed task IDs, files changed, checks and results, gate status, any spec edits, and remaining blockers. Confirm that the plan is updated.
5. Stop for feedback or explicit instruction to start the next phase. Address feedback within the current phase and rerun affected checks before presenting the updated review.

If validation fails or a task is blocked, keep the phase incomplete and record the cause in the plan’s Blockers section. Report whether it is a plan issue or requires a spec change; do not silently skip to a later task. Fix regressions within scope; do not weaken tests or acceptance criteria to obtain a pass. Escalate required scope changes using the Spec Change Escalation rule. Do not skip a failed or blocked phase to start later work.

### Stage 4: Verify feature completion

After the final phase, compare the implementation against every spec acceptance criterion. Report each criterion as met, unmet, or unverified, with supporting checks or evidence. Record remaining gaps and completion status in the plan and state whether the feature is ready for review or needs further work. When multiple plans exist, verify all referenced plans before reporting feature completion. Completed phases alone do not establish feature completion.

## Spec Lock Rule

Treat the spec as locked once the user approves it or asks for implementation without requesting requirement changes.

Do not start implementation while any `[NEEDS CLARIFICATION]` markers remain. Resolve them first, then treat the cleaned spec as the lockable source of truth.

During implementation, the user may change direction. Use judgment to decide whether the change is small or large:

**Small change** — absorb the user's requested change. Update the spec inline, report the affected requirement and any phase impact, and continue. A change is small if it:
- doesn't invalidate work already completed
- doesn't change the core approach or architecture
- can be absorbed into the current or next phase without restructuring

**Large change** — stop and escalate before touching anything. A change is large if it:
- invalidates completed work
- significantly expands or restructures scope
- changes the core approach or introduces new surfaces

Do not silently absorb a large change by treating it as a small one.

If the agent discovers a requirement change that the user has not requested, use the escalation rule before editing the locked spec, even if the change seems small.

## Spec Change Escalation

When a change is too large to absorb inline, say so explicitly before proceeding. Present:

1. What the user is asking for
2. Why it is a large change (what it invalidates or restructures)
3. Proposed spec revision
4. Revised implementation approach

Wait for user approval before editing the spec or continuing implementation.

## Choosing the Active Spec and Plan

Determine the active spec and its referenced plan in this order:

1. Explicit user-provided file or feature ID
2. Files referenced in the current task context
3. Files matching the current branch or active feature name
4. The only matching spec and plan in the repository

If multiple candidates remain, ask the user which to use.

## Guardrails

- Do not advance past the current phase without explicit user instruction.
- Do not silently change locked requirements by editing the spec during implementation.
- Do not guess which spec or plan is active when multiple candidates exist.
- Do not store execution state in the spec.
- Use the spec as the source of truth for requirements and the plan for execution state.
