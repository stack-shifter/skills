---
name: expressjs-rest-api
description: Designs and implements REST APIs using Express 5 with TypeScript, following the target repository's patterns first. Use this whenever the user wants to add or modify Express routes, controllers, middleware, validation, authentication, repositories, services, or runtime composition — even if they only ask for "a route", "a controller", "a handler", "validation", or "some middleware".
---

## Purpose

Use this skill to design and implement Express 5 REST APIs in a way that fits the target repository first.

When local patterns exist, use them as the source of truth. When they do not, use the references in this skill as portable patterns to generate a coherent structure rather than one-off route code.

This skill is about Express API structure and boundaries, not database design. Persistence guidance in this skill stops at the boundary:

- controllers and services should depend on repositories, not datastore clients
- repositories should be aggregated behind one context or composition object when multiple repositories are used together
- business and domain layers should not construct datastore-specific requests directly

Portable references live in `references/`. Load only the patterns needed for the task:

- `references/app-pattern.md` for Express app setup and middleware composition
- `references/route-pattern.md` for route definitions with validation and auth middleware
- `references/controller-pattern.md` for request handlers, error flow, and response shaping
- `references/middleware-pattern.md` for auth, validation, and global error middleware
- `references/model-pattern.md` for TypeScript types and Zod v4 validation schemas
- `references/error-pattern.md` for typed error classes and `ErrorResponseBuilder`
- `references/status-code-pattern.md` for the `StatusCode` enum
- `references/pagination-pattern.md` for cursor pagination interfaces and encoding
- `references/repository-pattern.md` only for repository boundary, repository interfaces, and aggregate context guidance

## Express 5 Key Behaviors

Express 5 requires **Node.js 18 or higher**.

For new projects, this skill's default baseline is **Node.js 24** unless the target repository already uses another supported version.

**Async error propagation** is automatic. Rejected promises and thrown errors inside `async` route handlers forward to the error handler without an explicit `next(error)`. Controllers may still use `try/catch` to handle named error types before that fallback when the repository already does so.

**Built-in body parsing** — no separate `body-parser` package needed:

```ts
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

**Error-handling middleware** must declare all four parameters `(err, req, res, next)` or Express treats it as regular middleware.

**Route path syntax changed** — review wildcard and optional segment syntax carefully when migrating Express 4 routes.

**`req.query` should not be treated as the destination for coerced values** — store validated query output on `res.locals` instead of mutating `req.query`.

When the target repository already validates only for rejection and continues reading from `req.body`, `req.params`, or `req.query`, preserve that pattern unless the user explicitly wants a validation-pipeline refactor.

**`req.body` is `undefined` until a body parser runs** — do not assume JSON payloads exist before `express.json()`.

## Environment Variable Strategy

New projects use Node.js's built-in `process.loadEnvFile()` instead of `dotenv`. Three environment files are created at project root:

| File | Purpose | Committed to git |
|---|---|---|
| `.env.example` | Documents all required variables with empty or example values | Yes |
| `.env.development` | Local development values | No |
| `.env.production` | Production values | No |

`.gitignore` must include:

```text
.env.development
.env.production
```

In a composition or bootstrap module such as `app.ts`, import first, then load the correct file based on `NODE_ENV` immediately after imports and before application setup:

```ts
import express from 'express';

if (process.env.NODE_ENV !== 'production') {
  process.loadEnvFile(`.env.${process.env.NODE_ENV || 'development'}`);
}
```

In production, environment variables are injected directly into the process. Do not attempt to load a file in production.

Avoid reading `process.env` at module top level inside imported files before `loadEnvFile()` runs. If a dependency needs env values during initialization, move that env read into a function, a factory, or the app bootstrap path.

When the target repository already relies on package scripts such as `node --env-file=.env.development ...`, preserve that approach instead of adding `process.loadEnvFile()` to the app bootstrap.

## Source of Truth

This skill works across repositories. Do not assume a specific folder layout exists.

Use this order:

1. the target repository's existing patterns and conventions
2. the skill's documented patterns in `references/`
3. the inline examples in this skill

If local code differs from the skill examples, follow local code and use the examples only as design guidance.

## Repository Discovery

Start every use of this skill with a short discovery pass before proposing code.

Inspect only the parts of the repository that matter for the request. Common places include:

- the Express app entry point or bootstrap module
- router modules
- controllers or route handlers
- middleware modules
- services
- repositories or data-access modules
- models and Zod schemas
- dependency wiring or composition modules
- error class hierarchy and shared response/status utilities
- pagination helpers and cursor utilities

Look for:

- how the app bootstrap is composed
- how routes are mounted and versioned
- whether controllers already exist or should be added
- whether validation middleware stores parsed output on `res.locals` or only rejects invalid input
- whether auth middleware already exists
- whether repositories and a repository context already exist
- whether shared services or mappers already exist for the resource area
- whether utilities already solve status codes, cursor parsing, or error handling
- style indicators such as JSDoc conventions, inline step comments, response builder usage, and route path composition

After discovery, choose one mode and state it:

- `Existing pattern mode`: extend the repository's existing abstractions
- `Pattern generation mode`: generate the missing reusable pattern because no relevant abstraction exists

## Existing Pattern Mode

Use this mode when the repo already has patterns such as:

- an app bootstrap with helmet, cors, trust proxy, and global error middleware
- router-based route definitions with inline middleware
- `RequestHandler`-style async controllers or equivalent handlers
- auth middleware for Cognito JWT or another existing auth strategy
- validation middleware for body, query, and params
- a 4-parameter global error handler
- services for business logic and external integrations
- repository classes behind a composition or context layer
- model modules with TypeScript types and Zod schemas
- typed error classes and a shared status-code utility
- dependency injection or composition root modules

Preserve local patterns even when they reflect older Express 4-era conventions. Only generate new patterns when the repository does not already establish a coherent approach.

Patterns worth preserving when present:

- routes mounted under `/v1` with full resource paths declared inside each router
- controller-local `try/catch` blocks that map known repository errors into `ErrorResponseBuilder` payloads
- validation middleware that rejects invalid input without storing parsed values on `res.locals`
- a shared repository context or composition object that owns resource repositories
- function JSDoc and concise step comments inside handlers, middleware, and repositories

## Pattern Generation Mode

Use this mode when the target repository does not already provide a reusable abstraction for one or more layers.

In this mode:

- use the references as guidance for the shape of the generated code, not as a requirement that exact files or names exist
- introduce the smallest reusable abstraction that makes future routes easier to add and maintain
- prefer generating a reusable app bootstrap, router, middleware, service, repository, or utility pattern over embedding everything in a single file
- keep generated names and folders aligned with the target repository's conventions

## Preferred Project Shape

Unless the target repository already has a different established layout, use a structure like:

```text
src/
├── app.ts
├── controllers/
├── data/
│   ├── context.ts
│   ├── repositories/
│   └── repository.interface.ts
├── dependencies/
├── middlewares/
├── models/
├── routes/
├── services/
└── utilities/
```

Treat this as a conceptual layout, not a hard requirement.

When adding a new resource, a good flow is:

1. model in a model module
2. route in a router module
3. controller in a controller or handler module
4. service method if logic is non-trivial
5. repository in a data-access layer
6. wire dependencies in a composition layer
7. mount the route in the app bootstrap

## Default Assumptions

- Node.js version baseline for new projects: 24
- Express version: 5.x (`express ~5.1.0`)
- Language: TypeScript; follow the repository's existing module system and compiler settings first
- Validation: Zod v4 (`zod/v4`)
- Auth: preserve the repository's existing auth strategy; if generating a new one, prefer reusable JWT/Cognito middleware over inline checks
- Security headers: `helmet`
- CORS: `cors` with `CORS_ORIGIN` env var, `*` fallback
- Environment loading: Node.js `process.loadEnvFile()` in the app bootstrap after imports when the repository does not already use another approach
- Module system: follow the repository first; ESM is common in modern Express 5 TypeScript projects

If the user gives constraints that conflict with these defaults, adapt and state the change.

## Modern Pattern Preferences

When the target repository does not already establish a conflicting pattern, prefer:

- validated request data stored on `res.locals.validated` instead of mutating `req.query`
- thin controllers that throw typed domain errors and let global error middleware map them to HTTP responses
- repository-backed pagination with a shared cursor utility rather than ad hoc pagination per route
- structured logging and graceful shutdown hooks for new apps
- the repo's existing dev runner first; otherwise use `tsx watch` for new TypeScript projects, `node --watch` for compiled JavaScript, and `nodemon` only when custom watch behavior is already part of the project

## Working Rules

1. Read the relevant local app bootstrap, router, controller, middleware, service, repository, and composition code before editing.
2. If the repository already has an app bootstrap pattern, extend it. If not, generate one reusable bootstrap path instead of wiring Express ad hoc in multiple files.
3. If the repository already has router modules, extend them. If not, generate reusable `Router` modules instead of putting all endpoints directly in the app bootstrap.
4. Keep controllers thin. Put validation in middleware, orchestration in controllers, and persistence behind repositories.
5. Reuse an existing composition pattern for dependencies when it exists. If it does not, generate a lightweight composition module rather than constructing shared clients in every controller.
6. Reuse a shared error and status-code pattern when one exists. If it does not, generate one shared pattern instead of repeating inline HTTP error payloads.
7. Keep persistence mechanics behind repositories and one aggregate repository context. Controllers and services should not construct raw datastore requests directly.
8. Keep generated names, route prefixes, and response shapes aligned with the target repository's conventions.
9. Match the repository's documentation style when adding code comments: preserve JSDoc and brief step comments when those are established locally.

## Environment File Templates

### `.env.example`

```dotenv
NODE_ENV=
PORT=3000
CORS_ORIGIN=

# Auth / identity
AWS_REGION=
COGNITO_USER_POOL_ID=
COGNITO_APP_CLIENT_IDS=

# App-specific dependencies
APP_DEPENDENCY_CONFIG=
```

### `.env.development`

```dotenv
NODE_ENV=development
PORT=3000
CORS_ORIGIN=http://localhost:5173
AWS_REGION=us-east-1
COGNITO_USER_POOL_ID=us-east-1_XXXXXXX
COGNITO_APP_CLIENT_IDS=your-client-id
APP_DEPENDENCY_CONFIG=local
```

## Response Style

When using this skill, produce:

1. a short statement of the route(s) or component being added
2. whether you are in `Existing pattern mode` or `Pattern generation mode`
3. the planned implementation shape in order: model → route → controller → service → repository → dependencies/composition → app mount
4. explicit assumptions where auth, validation, persistence boundary, or bootstrap wiring is ambiguous

Avoid restating generic Express or AWS documentation when a repository-specific code snippet would answer better.
