---
name: stack-storm-rest-api
description: Design and implement AWS CDK REST APIs using Storm infrastructure and project-local application structure. Use for API Gateway REST routes, Lambda handlers, Cognito authorization, request validation, middleware, runtime composition, reusable API constructs, and related scheduled jobs.
---

## Purpose

Deliver working REST API changes that fit the target project. Separate infrastructure wiring from application behavior: handlers compose middleware, controllers orchestrate requests, services wrap external integrations, and repositories isolate persistence. Keep database modeling and WebSockets outside this skill.

## Infrastructure and Application Structure

Use `@stack-shifter/storm-stack` for supported API, Lambda, and resource-import constructs. Add a compatible dependency using the project’s package manager and verify its installed exports. If package access is unavailable, report the blocker rather than silently generating replacement constructs.

Keep application code in the familiar project-local structure: `authorizedGroup(...)`, `withCommonMiddleware` / `withWriteMiddleware`, controllers, services, repositories, a shared repository context, and `app.ts`. Storm supplies infrastructure; it does not replace these application layers. Reuse library utilities only when their interfaces fit the application's contract.

This skill covers Storm infrastructure. Alternative API and Lambda constructs are outside its scope. Do not migrate an existing API to Storm without a migration request; preserve its application code and public contracts during any requested migration.

## Source of Truth

This skill is repository-aware, not file-path-bound.

Use this order:

1. explicit user requirements
2. the target repository’s current architecture, naming, and API contracts
3. the patterns and examples in this skill’s references

If the local code differs from this skill's examples, follow local code and use the examples only as design guidance.

## Repository Discovery

Start every use of this skill with a short discovery pass before proposing code.

Inspect package manifests, lockfiles, installed exports and types, and relevant tests first. Then inspect the parts of the repository that matter for the request. Common places include:

- `lib/`, `infra/`, or `packages/*/lib/constructs/`
- stack files such as `core-stack.ts` or route registration modules
- `src/handlers/`
- `src/controllers/`
- `src/middlewares/`
- `src/services/`
- `src/models/validation/`
- `src/data/`
- `src/app.ts` or existing `src/dependencies/` modules
- shared response and utility modules

Look for:

- how routes are registered in CDK
- which handler export name the stack expects
- whether handler files export a typed `HANDLER` registry constant
- whether a route composition abstraction already exists
- whether shared Lambda naming and environment helpers already exist
- which middleware chain handlers already follow
- whether controllers already exist or should be added
- whether repositories and a repository context already exist
- whether shared services or mappers already exist for the resource area
- whether utilities already solve validation, authorization, cursor parsing, or error handling
- whether scheduled jobs already use a reusable construct

After discovery, choose the appropriate approach and state it:

- `Existing pattern mode`: the repository already has abstractions worth extending
- `Pattern generation mode`: the repository is missing one or more pieces, so generate code that establishes the pattern cleanly

Use Storm for infrastructure covered by this skill. Separate library interfaces from project-local helpers; verify names and signatures before choosing an example.

## Existing Pattern Mode

Use the repository's existing abstractions directly:

- Storm `RestApi`
- Storm `LambdaNode`
- importer helpers for existing infrastructure
- response helpers such as `RestResult`
- repository context or runtime composition modules such as `src/app.ts`
- middleware factories and authorization helpers

Avoid hand-rolling raw API Gateway, Lambda, or runtime wiring unless the existing abstractions clearly cannot support the requirement.

## Pattern Generation Mode

Use this mode when the target repository has no reusable abstraction for one or more layers.

In this mode:

- use Storm’s supported constructs for new infrastructure; do not generate local copies of the library
- use application references as guidance for the shape of the code, not as a demand that exact filenames or classes exist
- create only the minimal new abstraction needed to keep the generated code coherent and reusable
- use the connected project-local examples for new application structure; introduce shared abstractions when repetition or requirements justify them
- keep generated names and folders aligned with the target repository's conventions

## Implement the Requested Behavior

1. Identify the method, path, inputs, success/error responses, authorization, and required resource access. Preserve approved contracts and resolve material gaps before affected work.
2. Extend existing route and Lambda wiring. Keep shared configuration centralized and grant only the access each route needs. Explicitly configure public exceptions to protected defaults.
3. Verify entry files and exported handler names. Registry constants reduce string duplication but do not prove an export exists. Keep new CDK imports free of runtime dependency initialization; use a dependency-free registry module when appropriate.
4. Compose normalization, parsing, authorization, validation, and error translation in the order required by the installed middleware version. Pass parsed validation outputs to application logic.
5. Keep orchestration separate from datastore requests. Reuse existing dependency modules, repository exports, services, and response helpers. For new structure, follow the connected project-local examples: common/write middleware, controllers, repositories grouped in a context, and `app.ts` composition. Preserve a different established structure rather than imposing a migration.
6. Preserve status codes, error bodies, headers, and pagination contracts. Do not replace an application's responses with library defaults silently.

Use API Gateway REST API and route-specific Lambda exports as the baseline when the project provides no convention. Use Cognito Gateway authentication with handler-level authorization where required, middleware request validation, and shared environment wiring as the baseline. Choose runtime settings from the project’s supported dependencies and requirements; do not hardcode dependency versions from examples.

## Project Shape

Use this connected project-local structure as the baseline when the repository does not already provide one. Follow the shared middleware, controller, repository context, and `app.ts` examples in the references. Preserve existing `dependencies/` composition modules when present:

```text
lib/
├── constructs/          # project-specific extensions and helpers
└── core-stack.ts

src/
├── controllers/
├── data/
│   ├── context.ts
│   └── repositories/
├── handlers/
├── middlewares/
├── models/
│   └── validation/
├── services/
├── utilities/
└── app.ts
```

Import Storm constructs from the package; `lib/constructs/` is for project-specific helpers or extensions, not copied library implementations. Keep the application layout below regardless of the infrastructure choice.

Treat this as a conceptual layout, not a hard requirement:

- stack modules register routes and shared route defaults
- handlers contain Middy-wrapped Lambda exports
- controllers contain request orchestration and response shaping
- repositories isolate persistence access
- `src/data/context.ts` or an equivalent module wires repositories into one shared repository context
- `src/services/` holds logger, storage, notification, mapper, and other integration-facing services
- `src/utilities/` holds response helpers, error types, status codes, cursor helpers, and shared pure functions
- `src/models/validation/` contains request schemas for middleware validation
- `src/app.ts` wires singleton clients, repository context, and services

## References

Load only what the task needs:

- `references/rest-api-pattern.md`: Storm route wiring, defaults, grants, and handler registries.
- `references/node-lambda-pattern.md`: Lambda configuration and version-dependent defaults.
- `references/importer-pattern.md`: existing resource references.
- `references/auth-pattern.md`: Gateway authentication, scopes, and application authorization.
- `references/middleware-pattern.md`: parsing, validation outputs, and failure translation.
- `references/response-pattern.md`: response contracts and library compatibility.
- `references/runtime-composition-pattern.md`: module-scope clients and application dependencies.
- `references/persistence-boundary-pattern.md`: repository isolation and shared context composition.
- `references/services-pattern.md`: integrations and mapping.
- `references/utilities-pattern.md`: shared errors and cursor handling.
- `references/schedule-pattern.md`: related background jobs when requested.
- `references/api-review.md`: completion checklist.

## Validate and Deliver

Read `references/api-review.md` before completion. Run relevant project tests and type checks; synthesize affected CDK stacks for infrastructure changes. Confirm handler wiring, failure responses, and permissions. Do not deploy as validation. Report unavailable checks as unverified, with the reason.

When working under `stack-spec-workflow`, use its active delivery spec and approved plan. Record completed tasks, validation, and blockers there without creating competing plans or additional approval gates.

Finish with a concise account of routes implemented, configuration or contract changes, validation results, and remaining assumptions or limitations. Include snippets only when useful; do not stop at snippets when implementation was requested.
