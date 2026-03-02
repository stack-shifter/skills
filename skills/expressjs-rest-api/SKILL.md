---
name: expressjs-rest-api
description: Designs and implements REST APIs using Express 5 with TypeScript, following the target repository's patterns first. Use this whenever the user wants to add or modify Express routes, controllers, middleware, validation, authentication, repositories, or services — even if they only ask for "a route", "a controller", "a handler", "validation", or "some middleware".
---

## Purpose

Use this skill to build Express 5 REST API endpoints in a way that matches the target repository first.

When local patterns exist, use them as the source of truth. When they do not, fall back to portable baseline patterns.

Portable fallback references live in `references/`. Read only the file needed for the task:

- `references/app-pattern.md` for Express app setup and middleware composition
- `references/route-pattern.md` for route definitions with validation and auth middleware
- `references/controller-pattern.md` for request handlers, error flow, and response shaping
- `references/middleware-pattern.md` for auth, validation, and global error middleware
- `references/repository-pattern.md` for DynamoDB repository classes
- `references/model-pattern.md` for TypeScript types and Zod v4 validation schemas
- `references/error-pattern.md` for typed error classes and `ErrorResponseBuilder`
- `references/pagination-pattern.md` for cursor pagination interfaces and encoding

## Express 5 Key Behaviors

Express 5 requires **Node.js 18 or higher**.

**Async error propagation** is automatic — rejected promises and thrown errors inside `async` route handlers forward to the error handler without an explicit `next(error)`. Controllers still use `try/catch` to handle named error types before that fallback.

**Built-in body parsing** — no separate `body-parser` package needed:

```ts
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

**Error-handling middleware** must declare all four parameters `(err, req, res, next)` or Express treats it as regular middleware.

## Portability Rule

This skill works across repositories. Do not assume a specific folder layout exists.

Source of truth order:

1. The target repository's existing patterns and conventions
2. The skill's documented fallback patterns in `references/`

If local code differs from the skill examples, follow local code and adapt output to match it.

## Repository Discovery

Start every use of this skill with a short discovery pass before proposing code.

Search for:

- Express app entry point (`src/app.ts` or `src/index.ts`)
- Route files in `src/routes/`
- Controllers in `src/controllers/`
- Middleware in `src/middlewares/`
- Services in `src/services/`
- Repositories in `src/data/`
- Models and Zod schemas in `src/models/`
- Dependency wiring in `src/dependencies/`
- Error class hierarchy in `src/utilities/errors.ts`

After discovery, choose one mode and state it:

- `Local pattern mode`: extend the repository's existing abstractions
- `Fallback mode`: generate a portable baseline because no relevant abstraction exists

## Local Pattern Mode

Use this mode when the repo already has:

- `src/app.ts` — Express setup with helmet, cors, trust proxy, and global error middleware
- `src/routes/*.route.ts` — Router-based route definitions with inline middleware
- `src/controllers/*.controller.ts` — `RequestHandler` async functions
- `src/middlewares/authorize.middleware.ts` — Cognito JWT auth
- `src/middlewares/validation.middleware.ts` — Zod body/query/params validation
- `src/middlewares/global-error.middleware.ts` — 4-parameter error handler
- `src/services/` — business logic and external service integrations
- `src/data/*.repository.ts` — DynamoDB repository classes
- `src/models/*.model.ts` — TypeScript types and Zod schemas
- `src/utilities/errors.ts` — typed error classes
- `src/dependencies/` — dependency injection composition root

## Preferred `src/` Structure

Unless the target repository already has a different established layout, use:

```text
src/
├── app.ts
├── controllers/
├── data/
├── dependencies/
│   ├── aws.deps.ts
│   └── project.deps.ts
├── middlewares/
├── models/
├── routes/
├── services/
└── utilities/
    ├── errors.ts
    ├── status-code.ts
    └── mappers/
```

When adding a new resource, follow this file flow:

1. model in `src/models/<resource>.model.ts`
2. route in `src/routes/<resource>.route.ts`
3. controller in `src/controllers/<resource>.controller.ts`
4. service method in `src/services/` if logic is non-trivial
5. repository in `src/data/` for datastore access
6. wire dependencies in `src/dependencies/aws.deps.ts`
7. mount route in `src/app.ts`

## Default Assumptions

- Express version: 5.x (`express ~5.1.0`)
- Language: TypeScript with `commonjs` module output
- Validation: Zod v4 (`zod/v4`)
- Auth: Cognito JWT validation via `aws-jwt-verify`
- Datastore: DynamoDB via `@aws-sdk/lib-dynamodb`
- Security headers: `helmet`
- CORS: `cors` with `CORS_ORIGIN` env var, `*` fallback
- Single-table DynamoDB design with `PK`/`SK` key pattern

If the user gives constraints that conflict with these defaults, adapt and state the change.

## Required Environment Variables

```dotenv
CORS_ORIGIN=http://localhost:3000
COGNITO_USER_POOL_ID=us-east-1_XXXXXXX
COGNITO_APP_CLIENT_IDS=clientId1,clientId2
AWS_REGION=us-east-1
DYNAMODB_TABLE=MyTableName

# Local development only
PORT=3000
AWS_PROFILE=my-profile
DYNAMODB_ENDPOINT=http://localhost:8000
```

## Response Style

When using this skill, produce:

1. A short statement of the route(s) or component being added
2. Whether you are in `Local pattern mode` or `Fallback mode`
3. Files in order: model → route → controller → service → repository → dependencies → app mount
4. Explicit assumptions where auth, table shape, or validation is ambiguous

Avoid restating generic Express or AWS documentation when a repository-specific code snippet would answer better.
