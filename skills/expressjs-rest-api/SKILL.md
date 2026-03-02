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
- `references/postgres-pattern.md` for Postgres repository classes using Drizzle ORM
- `references/model-pattern.md` for TypeScript types and Zod v4 validation schemas
- `references/error-pattern.md` for typed error classes and `ErrorResponseBuilder`
- `references/status-code-pattern.md` for the `StatusCode` enum
- `references/pagination-pattern.md` for cursor pagination interfaces and encoding

## Express 5 Key Behaviors

Express 5 requires **Node.js 24 or higher**.

**Async error propagation** is automatic — rejected promises and thrown errors inside `async` route handlers forward to the error handler without an explicit `next(error)`. Controllers still use `try/catch` to handle named error types before that fallback.

**Built-in body parsing** — no separate `body-parser` package needed:

```ts
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

**Error-handling middleware** must declare all four parameters `(err, req, res, next)` or Express treats it as regular middleware.

## Environment Variable Strategy

New projects use Node.js's built-in `process.loadEnvFile()` instead of `dotenv`. Three environment files are created at project root:

| File | Purpose | Committed to git |
|---|---|---|
| `.env.example` | Documents all required variables with empty or example values | Yes |
| `.env.development` | Local development values | No |
| `.env.production` | Production values | No |

`.gitignore` must include:

```
.env.development
.env.production
```

In `app.ts`, load the correct file based on `NODE_ENV` before any other code reads `process.env`:

```ts
if (process.env.NODE_ENV !== 'production') {
  process.loadEnvFile(`.env.${process.env.NODE_ENV || 'development'}`);
}
```

In production (e.g. ECS, Lambda, Fly.io), environment variables are injected directly into the process — no file is loaded. Do not attempt to load a file in production.

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
- Datastore indicators: DynamoDB SDK imports, Drizzle schema files, `DATABASE_URL` env var

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
- `src/data/*.repository.ts` — repository classes (DynamoDB or Postgres)
- `src/models/*.model.ts` — TypeScript types and Zod schemas
- `src/utilities/errors.ts` — typed error classes
- `src/utilities/status-code.ts` — `StatusCode` enum
- `src/dependencies/` — dependency injection composition root

## Preferred `src/` Structure

Unless the target repository already has a different established layout, use:

```text
src/
├── app.ts
├── controllers/
├── data/
│   ├── repository.interface.ts
│   ├── pagination.ts
│   ├── client.ts               ← Drizzle client (Postgres projects only)
│   └── schema.ts               ← Drizzle schema (Postgres projects only)
├── dependencies/
│   ├── aws.deps.ts             ← AWS clients: Cognito, DynamoDB, etc.
│   └── project.deps.ts         ← shared singletons: logger, repositories
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
6. wire dependencies — for DynamoDB: `src/dependencies/aws.deps.ts`; for Postgres: `src/data/client.ts` + `src/dependencies/project.deps.ts`
7. mount route in `src/app.ts`

## Default Assumptions

- Node.js version: 24
- Express version: 5.x (`express ~5.1.0`)
- Language: TypeScript with `commonjs` module output
- Validation: Zod v4 (`zod/v4`)
- Auth: Cognito JWT validation via `aws-jwt-verify`
- Datastore: ask the user — DynamoDB or Postgres are both supported
  - DynamoDB: `@aws-sdk/lib-dynamodb`, single-table design with `PK`/`SK` keys
  - Postgres: Drizzle ORM (`drizzle-orm`) with `postgres` driver for new projects; follow the existing ORM if one is already in use
- Security headers: `helmet`
- CORS: `cors` with `CORS_ORIGIN` env var, `*` fallback
- Environment loading: Node.js `process.loadEnvFile()` — no `dotenv` package

If the user gives constraints that conflict with these defaults, adapt and state the change.

## Environment File Templates

### `.env.example` (DynamoDB project)

```dotenv
NODE_ENV=
PORT=3000
CORS_ORIGIN=

# AWS
AWS_REGION=
COGNITO_USER_POOL_ID=
COGNITO_APP_CLIENT_IDS=

# DynamoDB
DYNAMODB_TABLE=

# Local development only
AWS_PROFILE=
DYNAMODB_ENDPOINT=
```

### `.env.example` (Postgres project)

```dotenv
NODE_ENV=
PORT=3000
CORS_ORIGIN=

# AWS (if using Cognito auth)
AWS_REGION=
COGNITO_USER_POOL_ID=
COGNITO_APP_CLIENT_IDS=

# Postgres
DATABASE_URL=
```

### `.env.development` (Postgres example)

```dotenv
NODE_ENV=development
PORT=3000
CORS_ORIGIN=http://localhost:5173
AWS_REGION=us-east-1
COGNITO_USER_POOL_ID=us-east-1_XXXXXXX
COGNITO_APP_CLIENT_IDS=your-client-id
DATABASE_URL=postgres://postgres:password@localhost:5432/myapp_dev
```

## Response Style

When using this skill, produce:

1. A short statement of the route(s) or component being added
2. Whether you are in `Local pattern mode` or `Fallback mode`
3. Which datastore is in use (DynamoDB or Postgres)
4. Files in order: model → route → controller → service → repository → dependencies → app mount
5. Explicit assumptions where auth, table/schema shape, or validation is ambiguous

Avoid restating generic Express or AWS documentation when a repository-specific code snippet would answer better.
