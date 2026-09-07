# API Review Checklist

Use before reporting completion. Mark applicable checks passed, failed, or unverified with a reason; mark irrelevant checks not applicable.

## Wiring and Infrastructure

- Storm dependency version and actual exported interfaces checked; no alternative construct properties or invented library helpers.
- Entry files exist and configured handler exports resolve to functions; infrastructure imports do not newly initialize runtime dependencies.
- Methods, paths, authorizers, scopes, request models, and parameters match the requested contract.
- Shared defaults and route overrides produce intended environment, bundling, and permissions. Public routes are deliberate; default grants do not leak unnecessary access.
- Configured stage name and optional logging, monitoring, metrics, and throttling settings synthesize as intended; existing stage names stay unchanged unless requested. Custom token-authorizer overrides match the authentication contract when used.
- Relevant type checks, tests, and affected CDK synthesis pass. No deployment is performed as validation.

## Application Behavior

- Happy path and relevant not-found/conflict cases return expected status, body, and headers.
- Missing/malformed bodies and invalid path/query inputs fail consistently; parsed schema values reach controller logic.
- Authentication and authorization failures prevent controller execution and writes.
- Parser and unexpected middleware errors use the agreed error contract without sensitive details.
- Resource access is route-specific; external dependencies are mocked in tests.
- Persistence remains behind repositories; multi-system partial failure/retry behavior is checked when relevant.

## Skill Scenarios

When evaluating changes to this skill, exercise:

1. An installed Storm project adding a protected read route: public exports, constructor defaults, read grant, validated inputs, and working handler wiring.
2. An existing Storm API adding a route: common/write middleware, authorization, controllers, dependency modules, repository context, and response contract retained.
3. A new project with Storm installed: compatible dependency and direct library constructs, with common/write middleware, controllers, repository context, and app composition for the required application behavior. No copied infrastructure library or unnecessary wrapper functions.

For each scenario inspect generated code and run available type checks, focused tests, and synthesis. Static review alone does not verify runtime behavior. Record missing tooling or dependencies explicitly.

## Handoff

- Summarize completed routes and meaningful configuration/contract changes.
- Report checks and limitations accurately, including unverified synthesis or execution.
- Update the active delivery plan when using stack-spec-workflow; do not introduce a competing plan or approval gate.

## Infrastructure Selection Checks

- With Storm installed, prefer its supported constructs and verify their signatures.
- Without Storm, extend existing project constructs or synthesize project-local constructs without adding a Storm dependency.
- Explicit user preferences win; do not migrate existing APIs automatically.
- Exercise both infrastructure paths with the same application behavior and permission checks.

- Without Storm, stack examples call local construct route methods; native Lambda, API, stage, and integration wiring stays inside constructs. Check the documented subset of defaults and grants rather than assuming full Storm compatibility.
