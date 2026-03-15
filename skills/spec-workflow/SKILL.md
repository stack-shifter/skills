---
name: spec-workflow
description: Drives a spec-first, phased execution workflow for feature work. Use whenever the user wants a feature spec, a phased implementation plan, locked requirements, a `docs/specs` workflow, or a "implement the next phase only" loop where the spec stays stable and one or more plan files track execution.
---

# Spec Workflow Skill

## Purpose

This skill keeps one stable spec per feature as the source of truth and drives implementation through one or more mutable phased plan files.

Use this model:

- **Spec = what**
- **Plan = how**

The spec should remain mostly locked after approval. The plan evolves as phases and tasks are completed.

## When to Use

Use this skill when the user wants any of the following:

- a spec-first workflow for a feature
- a stable requirements document plus an execution plan
- phased implementation with explicit task tracking
- a locked spec with mutable plan files
- a "do only the next phase" loop
- an agent workflow that updates plan progress after each implementation phase

Typical trigger phrases:

- `Spec: build ...`
- "create a spec and plan for this feature"
- "turn this into a phased implementation plan"
- "implement the next phase from the plan"
- "use the spec as locked requirements"
- "set this up in docs/specs"

Do not use this skill for trivial one-off edits unless the user explicitly asks for the workflow.

## Canonical Layout

Prefer the repository's existing equivalent spec-and-plan convention if one already exists.

If the repository does not already have an established location, use this layout:

```text
AGENTS.md
CLAUDE.md

docs/
  specs/
    010_task-management.spec.md
    plans/
      010_task-management.plan.md
```

Use a shared numeric prefix and stable slug for the feature across the spec and plan files.

If `CLAUDE.md` exists in the repository, treat it as an additional instruction source alongside `AGENTS.md`.

## Naming Guidance

Use clear, stable file names:

- `<id>_<slug>.spec.md`
- `<id>_<slug>.plan.md`

Prefer readable slugs such as:

- `010_task-management`
- `011_user-invitations`
- `012_billing-webhooks`

Do not use vague slugs such as `stuff`, `misc`, or `new-feature`.

## Core Model

### 1) Spec = source of truth

The spec defines:

- overview
- scope and non-scope
- requirements
- contracts, invariants, and rules
- acceptance criteria
- references to the plan file or files

The spec should not track task checkboxes or step-by-step execution status.

### 2) Plan = phased execution

The plan defines:

- phases
- atomic task IDs
- task checkboxes
- expected files to change
- validation commands to run after the phase's tasks are complete
- phase gates and acceptance checks

The plan is the primary mutable document during implementation.

### 3) One run = one phase

During implementation, one execution run should complete the active phase: the first phase that still has unchecked tasks.

Within that run:

- execute the active phase's unchecked tasks in order
- keep each task small enough to produce a reviewable diff
- do not pull in tasks from a later phase
- run validation after the full phase is complete
- stop at the phase boundary and wait for explicit user approval before starting the next phase

## Required File Contents

### Spec File

`docs/specs/<id>_<slug>.spec.md` should contain:

1. `Overview`
2. `Scope / Non-Scope`
3. `Requirements`
4. `Acceptance Criteria`
5. `Plan References`

The `Plan References` section should point to one or more files in `docs/specs/plans/`.

Minimal spec template:

```md
# <Feature Title>

## Overview

## Scope / Non-Scope

### In Scope

### Out of Scope

## Requirements

## Acceptance Criteria

## Plan References

- `docs/specs/plans/<id>_<slug>.plan.md`
```

### Plan File

`docs/specs/plans/<id>_<slug>.plan.md` should contain phased execution such as:

- `Phase 1 - Skeleton + Types`
- `Phase 2 - Validation + Errors`
- `Phase 3 - Core Logic`
- `Phase 4 - Adapters`
- `Phase 5 - Hardening`

Within each phase, include:

- atomic tasks with stable IDs such as `P1.T1`
- checkboxes
- expected files to change
- validation commands to run after the phase's tasks are complete
- a gate describing what must be true before the phase is considered complete

If the feature changes external behavior, installation steps, user workflows, or operator workflows, the final phase must include the appropriate documentation updates. Update the right user-facing docs for the repository, which may include `README.md`, feature docs, runbooks, or other established documentation surfaces. Read the current docs first, identify the affected sections, and edit only those sections. Do not rewrite the documentation wholesale.

Minimal plan template:

```md
# <Feature Title> Plan

## Phase 1 - <Phase Name>

### Tasks

- [ ] `P1.T1` <small, reviewable task>
- [ ] `P1.T2` <small, reviewable task>

### Expected Files

- `path/to/file.ts`

### Validation

- `npm test -- feature`
- `npm run typecheck`

### Gate

- <condition that must be true before this phase is complete>

## Phase 2 - <Phase Name>

### Tasks

- [ ] `P2.T1` <small, reviewable task>

### Expected Files

- `path/to/other-file.ts`

### Validation

- `npm test -- feature`

### Gate

- <condition that must be true before this phase is complete>

## Blockers

- None currently.
```

## Holistic Validation Pass

Before approving, revising, or executing a spec/plan pair, perform these checks against the repository:

1. Surface audit
   - search for every renamed or removed symbol, config key, route, UI string, contract, and behavior named in the spec
   - identify all impacted layers, not just the files already named in the docs
   - include any relevant application, UI, API, data, infrastructure, automation, test, and documentation surfaces

2. Scope consistency
   - check for contradictions between Overview, Scope, Out of Scope, Requirements, Contracts, Acceptance Criteria, and Plan References
   - if an acceptance criterion requires changing an out-of-scope layer, either narrow the acceptance criterion or bring that layer into scope

3. Phase executability
   - verify each phase can pass its listed validation commands using only work completed in that phase and earlier phases
   - reorder phases when build, type, test, validation, deployment, migration, or integration steps depend on later-phase changes

4. Explicit ownership
   - if a requirement says to update callers, consumers, adapters, views, handlers, controllers, services, tests, docs, configs, or similar downstream surfaces, name them explicitly in the plan tasks and expected-file summary
   - do not leave high-risk coupling as implied cleanup

5. Contract boundaries
   - distinguish product behavior, application/runtime behavior, storage/schema behavior, integration behavior, and infrastructure behavior
   - state clearly when a cleanup changes one boundary but intentionally leaves another unchanged

When reviewing, present findings in this order:

1. contradictions
2. missing implementation surface
3. phase or ordering risks
4. test or verification gaps
5. suggested doc fixes

## Choosing the Active Spec and Plan

When multiple feature specs or plans exist, determine the active files in this order:

1. explicit user-provided file or feature ID
2. files referenced in the current task context
3. files matching the current branch or active feature name
4. the only matching spec/plan pair in the repository

If multiple candidates still remain, do not guess. Ask the user which feature ID or file pair to use.

## Single Plan vs Multiple Plans

Use one plan file by default.

Use multiple plan files only when the feature has clearly separate execution tracks, such as:

- backend and frontend work that can progress independently
- application work plus infrastructure or migration work
- a large feature with separate rollout streams

If multiple plans exist:

- the spec must reference all of them
- each plan must own a distinct scope
- the current run should operate on only one active plan unless the user explicitly asks otherwise

## Workflow

### Phase A: Create or refine the spec

When no spec exists yet:

1. Gather context from the repo and the user request.
2. Run the holistic validation pass for the known feature surface so the draft spec reflects actual repo coupling.
3. Draft the spec file in the repository's established spec location, or in `docs/specs/` if no equivalent location exists.
4. Include plan references in the spec.
5. Ask the user to review and lock the spec before implementation.

When a spec already exists:

1. Treat it as the source of truth.
2. Do not modify it unless the user explicitly asks to revise requirements.
3. Still run the holistic validation pass before accepting the spec as execution-ready.

### Phase B: Generate the phased plan

Once the spec is approved or already locked:

1. Create or revise the plan file in the repository's established plan location, or in `docs/specs/plans/` if no equivalent location exists.
2. Run the holistic validation pass against the locked spec and current repo state before finalizing phase boundaries.
3. Break execution into phases.
4. Write atomic tasks with IDs and checkboxes.
5. Add expected files, validation commands, and gates.
6. Keep plan tasks concrete enough that a single phase run can complete them in order and produce a reviewable diff.

### Phase C: Implementation loop

For each execution run:

1. Read `AGENTS.md`.
2. Read `CLAUDE.md` if it exists.
3. Read the locked spec.
4. Read the active plan.
5. Identify the active phase: the first phase that still has unchecked tasks.
6. If any task in the active phase is underspecified, refine those plan tasks before coding.
7. Implement all unchecked tasks in the active phase, one at a time, in order.
8. Update each task checkbox as its implementation work is completed.
9. After all tasks in the phase are complete, run the phase's validation commands.
10. Run the full test suite.
11. Verify the phase gate.
12. Update the plan to reflect phase completion.
13. Present a phase review to the user.
14. Stop. Wait for the user to either provide feedback or explicitly approve moving to the next phase.

## Task Design Rules

Each task should usually fit inside one focused chunk of work within a phase run and produce one reviewable diff.

Good task shapes:

- add domain types for task entity
- add request validation for create-task route
- implement repository method for task lookup

Bad task shapes:

- build backend
- implement entire feature
- finish API and UI

If a plan contains oversized tasks, split them before implementation.

## Implementation Loop Rules

When operating in the implementation loop:

- execute all unchecked tasks within the active phase, in order
- do not pull in tasks from a later phase
- keep diffs minimal and scoped to the active phase
- use the spec as locked requirements
- prefer updating the plan over rewriting the spec
- run validation after all tasks in the active phase are complete, not after each individual task
- always run the full test suite after phase-scoped validation
- stop at the phase boundary and wait for explicit user instruction before starting the next phase

A phase is complete when all of its tasks are checked off and its gate condition is satisfied.

If the full suite has failures:

- fix the regression by correcting the implementation
- do not delete, skip, or modify failing tests just to make them pass
- do not modify the spec or restructure plan phases to work around the failure
- if the regression cannot be fixed within the current phase's scope, surface it to the user before proceeding

When a phase is complete, present the review in this format:

```md
## Phase <N> Complete

**Tasks completed:** <list with task IDs>
**Files changed:** <list>
**Validation:** <commands run with brief results>
**Gate:** <pass / fail with details>

---
Provide feedback to adjust this phase, or reply "proceed" to start Phase <N+1>.
```

If the user provides feedback, address it within the current phase and re-present the phase review when done. Do not advance until the user explicitly approves.

## Plan Refinement Rule

If the next unchecked task is underspecified, ambiguous, or too large, refine the plan first.

Allowed refinements include:

- clarifying the task wording
- splitting a task into smaller tasks
- adding expected files
- adding validation commands
- clarifying the phase gate

Do not hide major design changes inside plan refinement. If the requirements change, that is a spec issue.

## Spec Lock Rule

Treat the spec as locked once the user approves it or clearly treats it as the accepted requirements baseline.

Examples of lock signals:

- "approved"
- "treat this spec as final"
- "use this as the locked requirements"
- asking for plan generation or implementation against the current spec without requesting requirement changes

Do not modify a locked spec unless:

- the user explicitly asks to change requirements
- implementation reveals a true contradiction or impossibility that requires spec revision

If the spec must change, update the spec deliberately, then reconcile the plan to match it.

## Spec Change Escalation

During planning or implementation, if the agent discovers that the spec likely needs to change, it must say so explicitly before proceeding.

Examples:

- the spec conflicts with the actual repository constraints
- the spec is missing a requirement needed to complete the next task
- the spec contains an invalid assumption
- the accepted scope should be split, reduced, or expanded to remain coherent
- the current plan cannot continue without changing a locked requirement

When this happens, present the issue in this format:

1. `Affected section`
2. `Problem`
3. `Why discovered now`
4. `Proposed revision`
5. `Plan impact`

Wait for user approval before editing the spec.

Do not silently edit the spec just because implementation uncovered new information.

## Plan Mutation Rule

The plan is expected to change during the work.

Allowed plan updates include:

- checking off completed tasks
- clarifying task wording
- splitting a task into smaller tasks
- adding acceptance notes
- updating commands or file lists
- recording blockers

Do not convert the plan into a narrative status document. Keep it execution-oriented.

## Drift Reconciliation Rule

If the codebase has already drifted from the plan but not from the spec:

1. reconcile the plan to reflect reality
2. mark already completed work accurately
3. add or revise tasks so the remaining plan still matches the spec
4. then continue with the next unchecked task

Do not force implementation to follow an outdated plan when the spec remains correct.

## Blocker Handling

If a task within the active phase is blocked:

- do not silently skip to a later task
- record the blocker in the plan
- ask the user for direction if the blocker changes requirements or ordering
- do not advance to the next phase while a blocker is unresolved

When the next task is blocked, update the plan with a concise blocker note such as:

```md
- Blocked: waiting on schema decision for `status` enum
- Blocked: task depends on `P2.T3` requirement clarification
```

State:

- what is blocked
- why it is blocked
- whether the blocker is a plan issue or a spec issue

Do not skip forward unless the user explicitly approves resequencing.

## Completion Rule

When all tasks in the active plan are complete:

1. verify the implemented work against the spec acceptance criteria
2. identify any remaining gaps or follow-up work
3. update the plan to reflect completion accurately
4. report whether the feature appears ready for review, rollout, or another planning pass

Completion is defined by the spec and acceptance criteria, not just by checked boxes.

## Standard Operating Prompt

Use this instruction pattern when the user wants the loop:

> Use `@AGENTS.md` for constraints.
> Use `@CLAUDE.md` for additional constraints if it exists.
> Use the active spec file as LOCKED requirements and do not modify it unless explicitly instructed.
> Use the active plan file to identify the active phase (first phase with unchecked tasks).
> Implement all unchecked tasks in that phase, in order.
> After completing all tasks in the phase, run the phase's validation commands and then the full test suite.
> When the phase is complete, verify the phase gate, present the phase review, and invite feedback or a proceed instruction.
> Do not advance to the next phase until the user explicitly approves.

## Output Expectations

When drafting the workflow artifacts:

- clearly state which spec and plan files are being used
- identify whether you are creating the spec, generating the plan, refining the plan, or executing the active phase
- when implementing, name the selected phase and task IDs before making edits

When finishing an implementation run:

- report the phase and each task completed within it
- summarize files changed
- list commands run and results
- state whether the phase gate was satisfied
- confirm the plan was updated
- present the phase review prompt and explicitly invite feedback or a proceed instruction before stopping

## Guardrails

- Do not treat the spec and plan as interchangeable.
- Do not store execution checkboxes in the spec.
- Do not advance past the current phase without explicit user instruction.
- Do not implement tasks from a later phase in the same run as the active phase.
- Do not silently change locked requirements by editing the spec during implementation.
- Do not guess which spec or plan is active when multiple candidates exist.
- Prefer one spec per feature, with one or more plan files referenced from that spec only when needed.
