---
name: branch-risk-review
description: Compares the current branch to main to identify breaking changes, introduced bugs, security regressions, and operational or performance risks. Use when reviewing PRs, preparing releases, auditing branch readiness, or when asked “what changed vs main?”, “is this safe to merge?”, “what are the risks in this branch?”, or similar questions about branch impact versus a base branch.
---

## Purpose

This skill provides a structured, risk-focused comparison of the current branch (`HEAD`) against `main` (or an alternate base) to surface:
- Breaking changes (contracts, APIs, schemas, UI flows)
- Bug/regression risk
- Security/auth regressions
- Performance/cost/reliability impacts
- Operational/deployment risks
- Test/coverage gaps

The goal is not just to summarize the diff. The goal is to identify what changed, why it matters, how confident the analysis is, and what should be verified before merge or release.

## When to Use

Use this skill when the user asks to compare current work versus `main` (or a specified base), including:
- “Compare my branch to main”
- PR/release readiness reviews
- “What changed vs main?”
- “Are there breaking changes or new bugs?”

If the user specifies a different base (e.g., `develop`, release branch, tag), use that base instead of `main`.

## Defaults and Assumptions

- Default base branch: `main`
- Default head: current checked-out branch / `HEAD`
- If the repo uses `master` instead of `main`, detect and use the correct base.
- Assume a git repository is available unless stated otherwise.

## Output Format

The final output must include these sections:

1. Change Summary
2. Overall Risk
3. Top Findings
4. Breaking Changes
5. Regression / Bug Risks
6. Security / Auth Risks
7. Data / Migration Risks
8. Performance / Reliability Risks
9. Operational / Deployment Risks
10. Test & Validation Notes
11. Recommended Follow-ups

`Overall Risk` must be one of: `low`, `medium`, `high`

For each flagged item include:

- Severity: `high`, `medium`, or `low`
- Confidence: `high`, `medium`, or `low`
- Evidence: specific file paths, symbols, contracts, or tests where possible
- Confirmed change: what the diff definitely changed
- Predicted impact: what could fail or degrade because of that change
- Recommendation: a concrete next step such as a test, mitigation, rollout guard, or docs update

If no material risks are found, say so explicitly and still include residual uncertainty, review scope, and prudent follow-up verification.

## Workflow

### 1) Establish the Comparison Range

1. Fetch latest refs (without rewriting history):
   - `git fetch --all --prune`
2. Identify base:
   - Prefer `origin/main`, else `origin/master`, else local `main/master`.
3. Determine merge base:
   - `MERGE_BASE=$(git merge-base <base> HEAD)`

Use the merge base to scope diffs when appropriate.

### 2) Summarize What Changed

Provide a concise summary of:
- High-level file/category changes (API, DB, infra, auth, UI)
- Additions/removals/renames
- “Hot spots” (large diffs, core modules, shared libraries)

Recommended commands (use what’s available):
- `git diff --stat <base>...HEAD`
- `git diff --name-status <base>...HEAD`
- `git log --oneline --decorate <base>..HEAD`

If there are many files, prioritize review of:
- Shared libraries/utilities
- Auth/security modules
- Data model and migrations
- API endpoints / contracts
- Infra/deploy configs
- Core business logic paths

For very large diffs, explicitly state when the review is partial and name the areas prioritized. Do not imply exhaustive review if only hot spots were examined.

### 3) Classify Risk Areas

Explicitly evaluate and report in each section below.

Distinguish between:

- **Confirmed change**: what the diff definitely changed
- **Predicted risk**: what might fail because of that change

Do not present speculation as certainty. Lower confidence when the evidence is indirect, when runtime behavior depends on unreviewed code paths, or when tests are missing.

#### A) Breaking Changes

Flag as **Breaking** when changes may require coordinated updates to consumers/dependents.

Look for:
- Public API route changes (path/method), request/response schema changes
- Contract changes: event payloads, message formats, external integration contracts
- Renamed/removed exported symbols; changed function signatures/interfaces
- UI flow changes that break deep links or expected navigation
- CLI command/flag changes
- Config keys changed/removed; default behavior changes
- Non-backward-compatible schema/migration changes (drops/renames, type narrowing)

For each finding, include:
- What changed
- Who/what is impacted
- Compatibility path (versioning, fallback, dual-write, feature flags)

#### B) Bug / Regression Risk

Flag as **Regression Risk** when changes may introduce functional errors.

Look for:
- Logic changes without corresponding test changes
- Conditional inversions, null-handling changes, default value changes
- Concurrency/async changes (races, missing awaits, cancellation issues)
- Error handling changes (swallowed exceptions, retry/timeout behavior changes)
- Boundary condition changes

For each finding, include:
- Suspected failure modes
- Where it manifests (user-facing vs internal)
- Tests/scenarios that should cover it

#### C) Security / Auth Regressions

Flag as **Security Risk** when changes affect identity, authorization, secrets, or trust boundaries.

Look for:
- AuthZ logic changes; role/scope checks added/removed
- New endpoints or broadened access
- Secrets handling changes (env vars, config files, logs)
- Input validation changes (schema validation, sanitization)
- Dependency updates with potential security impact (lockfile changes)

For each finding, include:
- Change type (authn/authz/data exposure)
- Potential impact
- Mitigations (least privilege, validation, logging hygiene)

#### D) Data / Persistence Risk

Flag as **Data Risk** when changes affect storage, schema, migrations, indexing, or lifecycle.

Look for:
- Destructive or non-idempotent migrations; ordering issues
- Model changes requiring backfills
- Index additions/removals
- Serialization/deserialization changes

For each finding, include:
- Backward/forward compatibility notes
- Migration safety (reversible? staged rollout? verified?)

#### E) Performance / Cost / Reliability Risk

Flag as **Perf/Cost Risk** when changes impact hot paths or resource usage.

Look for:
- N+1 queries, unbounded loops, payload size increases
- Logging verbosity increases in hot paths
- Caching changes
- Timeout/retry/circuit-breaker modifications
- Scaling defaults changed

For each finding, include:
- Likely bottlenecks
- Benchmarks/load tests to run
- Observability needed to validate

#### F) Operational / Deployment Risk

Flag as **Ops Risk** when changes affect deployment, configuration, or runtime assumptions.

Look for:
- IaC changes, pipeline changes, base image/runtime updates
- New required env vars/secrets
- Breaking config changes
- Health checks/readiness changes

For each finding, include:
- Rollout considerations
- Rollback difficulty
- Required release notes/runbook updates

### 4) Validate Test and Coverage Signals

Report at minimum:
- Whether tests were added/updated with logic changes
- Skipped/disabled tests or removed assertions
- Test runner/config changes that could mask failures

Heuristics:
- High risk: core logic changed and no test changes
- Medium risk: tests exist but don’t cover the changed pathways
- Lower risk: tests expanded and include negative cases

Also call out when existing tests appear to cover the change well. This skill should report both confidence-building evidence and gaps.

### 5) Produce a Risk Review, Not Just a Diff Review

Prefer concrete, evidence-backed findings over generic warnings.

Good:

- "Route response schema changed in `src/api/orders.ts`; contract test was not updated, so clients relying on `status` may break."

Bad:

- "API changes may cause regressions."

When evidence is insufficient, say what you found and what remains uncertain.

### 6) No-Findings Behavior

If no material risks are identified:

- state `No material risks found from the reviewed diff`
- mention review scope
- mention residual uncertainty, if any
- list any missing verification work that would still be prudent before merge or release

## Optional Enhancements

Use when helpful:
- Propose changelog/release note bullets if the repo has a convention
- If SemVer applies, recommend major/minor/patch impact
- For migrations, propose a safe rollout sequence (expand/contract, dual-write, backfill)

## Guardrails

- Do not label something “breaking” without pointing to a specific contract surface (API/interface/schema/config).
- Do not assert “bug introduced” as fact unless a concrete defect is evident; frame as risk.
- Prefer high-signal findings over exhaustive line-by-line commentary; prioritize shared/public surfaces and production paths.
- Separate confirmed change from predicted failure mode so the reader can tell evidence from inference.
