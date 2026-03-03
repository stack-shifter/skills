---
name: cdk-rest-api-postgres
description: Designs and implements REST APIs on AWS using this repository's CDK patterns first, with AWS Lambda handlers and Postgres via Drizzle as the only datastore. Use this whenever the user wants to add or modify API Gateway REST endpoints, Lambda handlers, Cognito auth, request models, reusable CDK constructs, Drizzle schema, SQL repositories, or database-backed CRUD routes in this project, even if they only ask for "an endpoint", "a handler", "a table", "a repository", or "some CDK wiring".
---

## Purpose

Use this skill to compose API endpoints the way this repository already does it.

Treat Postgres plus Drizzle as the source of truth for persistence work. When the local CDK constructs and runtime patterns exist, extend them instead of generating generic AWS examples.

Portable repository-specific references live in `references/`. Read only the file needed for the task:

- `references/rest-api-pattern.md` for route composition through this repo's CDK patterns
- `references/node-lambda-pattern.md` for Lambda defaults and environment wiring
- `references/importer-pattern.md` for importing existing AWS resources into the stack
- `references/response-pattern.md` for controller and middleware response conventions
- `references/sql-drizzle-pattern.md` for Drizzle schema, repository, and `DatabaseContext` guidance
- `references/auth-pattern.md` for Cognito authorizer, scopes, and handler-level group authorization

## Repository Rule

This skill is repository-first, not generic-first.

The source of truth order is:

1. The target repository's current constructs, stack code, handlers, middlewares, and data layer
2. The guidance in this skill

If the local code differs from examples in this skill, follow local code and adapt the output to match it.

## Repository Discovery

Start every use of this skill with a short discovery pass before proposing code.

Read only the files needed for the request, usually from these locations:

- `lib/core-stack.ts`
- `lib/constructs/rest-api.ts`
- `lib/constructs/node-lambda.ts`
- `lib/constructs/api-models.ts`
- `src/handlers/`
- `src/controllers/`
- `src/middlewares/`
- `src/models/validation/`
- `src/data/db/`
- `src/data/repositories/`
- `src/utilities/rest-result.ts`
- `src/app.ts`

Look for:

- how the route is registered in CDK
- which handler export name the stack expects
- which Middy middleware chain the handler follows
- whether the controller already exists or should be added
- whether a repository or schema already models the data
- whether the change needs environment variables from `getDefaultLambdaEnvironment`

After discovery, work in `Local construct mode`.

## Local Construct Mode

Use the repository's existing abstractions directly:

- `RestServerlessApi` for API Gateway REST route composition
- `NodeLambda` for Lambda defaults and bundling
- `Importer` when the stack needs to attach to existing AWS resources
- `RestResult` for API Gateway response objects
- `DatabaseContext` plus Drizzle repositories for persistence

Avoid hand-rolling raw API Gateway, Lambda, or persistence wiring unless the existing abstractions clearly cannot support the requirement.

## Working Rules

1. Read the relevant local construct, stack, handler, controller, and repository code before editing.
2. Build REST endpoints through `RestServerlessApi`; do not replace local route composition with raw CDK unless necessary.
3. Keep handlers thin. Put orchestration in controllers and persistence in `src/data/repositories/`.
4. For database work, model schema in Drizzle and query via the shared `db` client.
5. Reuse `DatabaseContext` from `src/app.ts` instead of creating ad hoc database clients in handlers.
6. Reuse the local Middy middleware stack pattern: header normalization, event normalization, optional JSON parsing, auth middleware, validation middleware, and shared HTTP error handling.
7. Return responses through `RestResult`, including `RestResult.fromDatabaseError(...)` for recognized SQL constraint failures.
8. Preserve Cognito authorizer behavior and group-based authorization middleware when extending protected routes.

## Default Assumptions

- API type: API Gateway REST API
- Compute: one Lambda handler export per route
- Datastore: Postgres via Drizzle and the shared Neon HTTP client
- Runtime defaults: `NodeLambda` defaults from this repo
- Auth: Cognito authorizer at API Gateway plus `authorizedGroup(...)` in handlers when needed
- Validation: Zod schemas through `validation.middleware.ts`
- Environment: Lambda runtime receives `DATABASE_URL` and shared app settings from `getDefaultLambdaEnvironment()`

If the user gives constraints that conflict with these defaults, adapt and state the change.

## Project Shape

Follow this repository structure:

```text
lib/
├── constructs/
└── core-stack.ts

src/
├── controllers/
├── data/
│   ├── db/
│   │   ├── client.ts
│   │   └── schema/
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

Use it this way:

- `lib/core-stack.ts` registers routes and shared route defaults
- `src/handlers/` contains Middy-wrapped Lambda exports
- `src/controllers/` contains request orchestration and response shaping
- `src/data/db/schema/` defines Drizzle Postgres tables and relations
- `src/data/repositories/` contains SQL access logic
- `src/data/context.ts` wires repositories to a shared `Db`
- `src/models/validation/` contains Zod schemas for request validation

## Construct-Centered Guidance

### 1. `RestServerlessApi` owns route composition

Register routes in `lib/core-stack.ts` through:

- `get`
- `getById`
- `post`
- `put`
- `delete`

Set shared defaults once:

```ts
const cognitoAuthorizer = api.createCognitoAuthorizer(userPool);

api.setDefaultRouteOptions({
    authorizer: cognitoAuthorizer,
    environmentVariables: getDefaultLambdaEnvironment(),
});
```

Important details from this repo:

- `routePath` must start with `/`
- default route options merge with per-route options
- setting `authorizer: undefined` on a route removes the default authorizer
- scopes are passed per route in `lib/core-stack.ts`
- route grants are connection-based; no table-level IAM grants are needed for SQL persistence

### 2. `NodeLambda` provides runtime defaults

This repo's Lambda defaults include:

- Node.js 24.x
- ARM64
- 256 MB memory unless overridden
- 15 second timeout unless overridden
- active X-Ray tracing
- CloudWatch log group with three-month retention
- bundling with `@aws-sdk/*` externalized

`getDefaultLambdaEnvironment()` is part of the runtime contract. It currently requires:

- `DATABASE_URL`
- `FRONTEND_URL`
- `S3_BUCKET`
- `SES_IDENTITY_EMAIL`
- `SES_IDENTITY_EMAIL_ARN`
- optional `CORS_ORIGIN`
- optional `ADMIN_GROUP`

When adding routes, inherit that shared environment unless there is a strong reason not to.

### 3. `CoreStack` is the route composition source of truth

Route wiring in this repo follows a stable pattern:

```ts
api.get({
    routePath: "/clients",
    lambdaName: lambdaName("ClientsQuery"),
    filePath: handlerPath("src/handlers/client.handler.ts"),
    handlerName: "queryClientHandler",
    description: "/clients",
    scopes: readScopes,
});
```

Prefer:

- one handler file per resource area
- exported handler name constants when they already exist
- `handlerPath(...)` and `lambdaName(...)` helpers from `lib/core-stack.ts`
- shared Cognito authorizer defaults with per-route scopes

## Runtime Guidance

### Handlers

Handlers should follow the local Middy pattern:

- `httpHeaderNormalizer()`
- `httpEventNormalizer()`
- `handleHttpError()`
- `httpJsonBodyParser({ disableContentTypeError: true })` for write routes
- `authorizedGroup(...)` when group authorization is required
- `validateHeaders(...)`, `validatePathParameters(...)`, `validateQueryParameters(...)`, and `validateBody(...)` as needed

Keep handler files mostly declarative. They should compose middleware around controller functions rather than implement business logic directly.

### Controllers

Controllers should:

- read typed event data after middleware validation
- call `dbContext` and other shared services from `src/app.ts`
- map entities through the repository and mapper layers
- return `RestResult` responses
- translate repository failures with `RestResult.fromDatabaseError(...)` when applicable

### Data Layer

Persistence work belongs under `src/data/`.

Preferred flow:

1. define or update table schema in `src/data/db/schema/`
2. export it through the schema index if needed
3. implement repository behavior in `src/data/repositories/`
4. wire the repository through `src/data/context.ts` if it is a new repository
5. consume it from controllers via `dbContext`

Do not open raw database connections inside handlers or controllers. Reuse the shared `db` and `DatabaseContext`.

### Drizzle Schema Rules

When adding or changing persistence:

- use `pgTable`, typed columns, indexes, enums, and relations from Drizzle
- keep relations centralized in `src/data/db/schema/relations.schema.ts` when appropriate
- mirror existing schema naming and table file conventions
- prefer SQL constraints and indexes over application-only uniqueness logic
- generate migrations with the repo's Drizzle workflow instead of inventing manual SQL files

### Repository Rules

Repository APIs in this repo are SQL-oriented, not key-value oriented.

Prefer methods that reflect domain behavior, for example:

- `getById`
- `query`
- `save`
- `updateById`
- `delete`
- relation-specific helpers when needed

For cursor pagination, do not assume UUID v4 primary keys are time-ordered. Prefer a stable ordered tuple such as `created_at` plus `id`, and make the cursor carry both values.

## Response and Error Rules

Use `src/utilities/rest-result.ts` for all API responses.

Prefer:

- `RestResult.Ok(...)`
- `RestResult.Created(...)`
- `RestResult.NoContent()`
- `RestResult.BadRequest(...)`
- `RestResult.NotFound(...)`
- `RestResult.Unauthorized(...)`
- `RestResult.Forbidden(...)`
- `RestResult.InternalServerError(...)`

For recognized Postgres constraint failures, prefer `RestResult.fromDatabaseError(error)` before falling back to generic 500 handling.

## Auth Rules

This repo protects routes in two layers:

1. API Gateway Cognito authorizer and scopes in CDK
2. handler-level group authorization via `authorizedGroup(...)`

When changing protected routes:

- preserve route scopes in `lib/core-stack.ts`
- preserve or extend group checks in handlers
- use `ADMIN_GROUP` semantics already present in the repo
- do not reintroduce removed client-account auth flows

## Change Patterns

When adding a new CRUD endpoint in this repo, the usual path is:

1. add or update the Drizzle schema if persistence changes
2. add or extend the repository
3. add or extend the controller
4. add or extend the handler with Middy middleware and validation
5. register the route in `lib/core-stack.ts`
6. add focused Jest coverage under `test/`

When the request is only infrastructure-facing, still confirm whether handler, controller, repository, validation, or environment wiring also needs to move with it.

## Definition of Done

A change using this skill is usually complete when:

- the route is registered through the local CDK abstractions
- the handler matches the repository's Middy style
- controller and repository responsibilities stay separated
- persistence uses Drizzle and Postgres only
- tests cover the changed behavior
