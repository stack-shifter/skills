---
name: approval-spec-workflow
description: Enforces an approval-gated, two-phase Spec workflow. Use when a prompt starts with "Spec:" or equivalent phrasing such as "spec:", "SPEC:", "write a spec first", "plan this before coding", or any explicit request to define the approach before implementation. Produce a detailed spec inline in the terminal, then implement only after explicit approval.
---

# Spec Workflow Skill

## When to Use

Trigger this workflow when the user explicitly asks for a spec-first or plan-first workflow, including:

- prompts beginning with `Spec:`, `spec:`, or `SPEC:`
- requests such as "write a spec first"
- requests such as "plan this before coding"
- any explicit instruction to present the approach for approval before implementation

Otherwise, proceed with normal execution.

## Scope Threshold

Default to this workflow for changes that are cross-file, high-risk, user-visible, architectural, operational, or likely to benefit from review before implementation.

Do not force a full spec for clearly trivial, low-risk work unless the user explicitly requests Spec mode.

## Core Principle

Always separate work into two phases:

- **Phase 1: Spec (no implementation)**
- **Phase 2: Implementation (only after approval)**

The spec is **always displayed inline in the command output**. Do not write spec files.

---

# Phase 1 — Spec Only (No Implementation)

## Phase 1 Hard Rules (Strict)

During Phase 1, the assistant must not:

- Edit or create any project files (source, config, docs, scripts, tests, lockfiles).
- Apply formatting-only changes, scaffolding, or “helpful” refactors.
- Add/remove dependencies or run migrations.
- Generate code, stubs, or patches.

If additional context is required, only read/inspect existing materials and reference them by file path in the spec.

Allowed during Phase 1:

- reading files
- searching the repository
- running non-mutating inspection commands
- running read-only checks that do not modify files or environment state

## Spec Output Requirements

Keep the spec concise by default. Aim for a short, reviewable spec. Expand only for high-risk, cross-cutting, or ambiguous work.

Output a Markdown-formatted specification in the terminal with the following sections (keep concise; expand only if high-risk):

1. **Problem Statement**

- What needs to be solved and why?

2. **Goals / Non-Goals**

- Goals: explicitly in-scope outcomes
- Non-goals: explicitly out-of-scope items

3. **Assumptions, Constraints, Dependencies**

- System constraints, organizational constraints, external dependencies.

4. **Proposed Approach (High-Level Design)**

- Components, responsibilities, execution flow.
- Boundaries and interfaces.

5. **Alternatives and Tradeoffs**

- At least **two alternatives**.
- For each: key tradeoffs (cost, complexity, risk, time, maintainability).

6. **Interface / Contract Changes (if any)**

- APIs, events, schemas, CLI contracts, UI flows, integrations.

7. **Implementation Plan**

- Small, reviewable steps.
- Prefer risk-reducing order; note migrations/backwards-compat if relevant.

8. **Test Plan**

- Unit/integration/e2e as applicable.
- Include edge cases and negative tests when relevant.

9. **Operations (as applicable)**

- Rollout steps, rollback approach, observability (logs/metrics/traces/alerts), runbooks.

10. **Risks & Questions**

- Key risks with mitigations.
- Open questions that need clarification.

## Clarifications

If requirements are ambiguous, ask focused clarifying questions and revise the spec accordingly.

If there is enough context to produce a useful first-pass spec, do that instead of blocking on questions. Put unresolved items in `Risks & Questions`.

---

# Approval Gate (Required)

After producing the spec, the assistant must stop and ask exactly:

`Approve: yes` or `Approve: no`

- If the user replies **Approve: yes**: proceed to Phase 2.
- If the user replies **Approve: no**: ask what to change, revise the spec (still Phase 1), then request approval again.

Treat the following as explicit approval equivalents:

- the exact reply `Approve: yes`
- a follow-up prompt that clearly includes an approval marker such as `(Approved)`
- a direct instruction to proceed with implementation after the spec has been reviewed

If the user response is ambiguous, do not implement yet. Ask for explicit approval.

## Auto-Accept / Non-Interactive Environments

If the environment may apply edits automatically, preserve the approval gate using a two-step invocation:

1. **Spec-only invocation** (the current message begins with `Spec:`):

- Output the spec only.
- Do not implement.
- End with: `To implement, re-run the command with "(Approved)" appended to the prompt.`

2. **Implementation invocation** (a follow-up user message):

- Only implement when the user re-invokes with an explicit approval marker, e.g.:
  - `Spec: <same task> (Approved)`

If the follow-up differs materially from the spec, treat it as a new Spec request and repeat Phase 1.

**STOP CONDITION:** After printing the spec and approval prompt, stop—do not implement anything in the same run unless an explicit approval marker is present.

---

# Trivial Task Exception

If the request is objectively trivial (e.g., typo fix, change a single constant with no functional impact), the assistant may ask:

`This appears trivial—proceed without spec? Reply "Approve: yes" to proceed or "Approve: no" to produce a spec.`

- If **Approve: yes**: proceed directly to implementation.
- If **Approve: no**: run the full Spec workflow.

---

# Phase 2 — Implementation (Only After Approval)

## Implementation Rules

Once approved:

- Implement strictly according to the approved spec.
- Prefer incremental changes that maintain a working state.
- Align outputs with the approved scope and constraints.
- Call out any material deviation from the approved spec before or during implementation.

## Handling Deviations

If deviations are discovered during implementation:

- **Minor deviations** (do not change core approach/assumptions/contracts/risk profile):
  - Continue implementation.
  - Report the delta in the output under: `## Implementation Updates`

- **Major deviations** (change core assumptions, approach, contracts, or risk profile):
  - Stop implementation.
  - Return to Phase 1, update the spec, and request re-approval.

## Completion Output

When implementation is complete, summarize:

- What changed (by area/component)
- Tests executed (and results)
- Operational/rollout notes and rollback notes (if applicable)
- Follow-up recommendations (if any)
