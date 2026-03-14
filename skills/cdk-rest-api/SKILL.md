---
name: cdk-rest-api
description: Designs and implements REST APIs on AWS using the target repository's CDK patterns first. Use this whenever the user wants to add or modify API Gateway REST endpoints, Lambda handlers, Cognito auth, middleware, request models, reusable CDK constructs, runtime composition, scheduled jobs, or repository-backed CRUD routes, even if they only ask for "an endpoint", "a handler", "some CDK wiring", or "an API route".
---

## Purpose

Use this skill to design and implement CDK-based REST APIs on AWS in a way that fits the target repository.

This skill is about API and platform patterns, not database design. Persistence guidance in this skill stops at the boundary:

- route handlers and controllers should depend on repositories
- repositories should be aggregated behind one repository context when multiple repositories are used together
- business and domain layers should not construct datastore-specific requests directly

Portable references live in `references/`. Load only the patterns needed for the task:

- `references/rest-api-pattern.md` for centralized API Gateway REST route composition
- `references/node-lambda-pattern.md` for Lambda defaults, naming, and shared environment wiring
- `references/importer-pattern.md` for importing existing AWS resources into CDK stacks
- `references/response-pattern.md` for shared API response helpers
- `references/auth-pattern.md` for Cognito authorizers, scopes, and the exception path for in-Lambda JWT verification
- `references/runtime-composition-pattern.md` for `src/app.ts`-style composition and repository context wiring
- `references/persistence-boundary-pattern.md` for repository pattern, aggregate repository context, and persistence isolation rules
- `references/middleware-pattern.md` for reusable Middy middleware such as auth, validation, and HTTP error translation
- `references/services-pattern.md` for logger, storage, notification, and mapper service design
- `references/utilities-pattern.md` for shared response, error, and cursor helpers
- `references/schedule-pattern.md` for EventBridge-triggered scheduled Lambda jobs

## Source of Truth

This skill is repository-aware, not file-path-bound.

Use this order:

1. the target repository's current architecture, naming, and abstractions
2. the patterns in this skill's references
3. the inline examples in this skill

If the local code differs from this skill's examples, follow local code and use the examples only as design guidance.

## Repository Discovery

Start every use of this skill with a short discovery pass before proposing code.

Inspect only the parts of the repository that matter for the request. Common places include:

- `lib/`, `infra/`, or `packages/*/lib/constructs/`
- stack files such as `core-stack.ts` or route registration modules
- `src/handlers/`
- `src/controllers/`
- `src/middlewares/`
- `src/services/`
- `src/models/validation/`
- `src/data/`
- `src/app.ts`
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

## Existing Pattern Mode

Use the repository's existing abstractions directly:

- route composition constructs such as `RestServerlessApi`
- Lambda wrappers such as `NodeLambda`
- importer helpers for existing infrastructure
- response helpers such as `RestResult`
- repository context or runtime composition modules such as `src/app.ts`
- middleware factories and authorization helpers

Avoid hand-rolling raw API Gateway, Lambda, or runtime wiring unless the existing abstractions clearly cannot support the requirement.

## Pattern Generation Mode

Use this mode when the target repository has no reusable abstraction for one or more layers.

In this mode:

- use the references as guidance for the shape of the code, not as a demand that exact filenames or classes exist
- create only the minimal new abstraction needed to keep the generated code coherent and reusable
- prefer introducing a small reusable construct, helper, middleware, repository context, or service pattern over shipping one-off route code
- keep generated names and folders aligned with the target repository's conventions

## Working Rules

1. Read the relevant local construct, stack, handler, controller, and repository code before editing.
2. If the repository already has a route composition abstraction, use it. If not, generate a small reusable pattern instead of scattering raw CDK logic.
3. Keep handlers thin. Put orchestration in controllers and persistence behind repositories.
4. Reuse an existing dependency composition pattern such as `src/app.ts` or a repository context when it exists. If it does not, create a lightweight equivalent rather than wiring dependencies ad hoc in handlers.
5. Reuse the local Middy middleware stack pattern when it exists. If it does not, generate a reusable middleware composition pattern instead of inlining validation and auth in every handler.
6. Return responses through the local response helper when one exists. Otherwise generate one shared response utility instead of repeating inline response objects.
7. Preserve Cognito authorizer behavior, scopes, and group-based authorization middleware when extending protected routes.
8. Prefer service classes and mapper classes for cross-cutting logic that appears in more than one controller.
9. Centralize reusable error and cursor helpers under shared utilities rather than duplicating them in controllers or handlers.
10. Keep persistence mechanics behind repositories and one aggregate repository context. Controllers and services should not construct raw datastore requests directly.

## Default Assumptions

- API type: API Gateway REST API
- Compute: one Lambda handler export per route unless the repository clearly uses another pattern
- Runtime defaults: existing Lambda wrapper defaults when present, otherwise a consistent Node.js Lambda baseline
- Auth: Cognito authorizer at API Gateway plus handler-level authorization middleware when needed
- Validation: middleware-level request validation
- Environment: Lambda runtime should receive shared app settings through one central helper or wiring layer when possible

If the user gives constraints that conflict with these defaults, adapt and state the change.

## Project Shape

Use a structure like this when the repository does not already provide a better one:

```text
lib/
├── constructs/
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

## Response Style

When using this skill, produce:

1. a short statement of the route or API capability being added or changed
2. a note saying whether you are using `Existing pattern mode` or `Pattern generation mode`
3. CDK route and Lambda snippets based on the local constructs when present, or the portable baseline when not
4. the repository, repository-context, controller, handler, middleware, and service follow-on work needed to make the route functional
5. explicit assumptions where auth, validation, or runtime wiring is ambiguous

Avoid datastore-specific design guidance unless the user separately asked for a database design skill.
