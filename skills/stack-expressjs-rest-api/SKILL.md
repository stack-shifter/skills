---
name: stack-expressjs-rest-api
description: Design and implement Express 5 REST APIs with TypeScript, following existing project patterns. Use for routes, controllers, middleware, validation, authentication, services, repositories, runtime composition, and optional Lambda Web Functions deployment.
---

## Purpose

Deliver working Express API changes using this order of precedence:

1. explicit user requirements
2. existing project architecture, naming, API contracts, and module settings
3. this skill's examples

Keep examples and generated code simple: arrow functions, plain `RequestHandler` declarations, named DTOs on local variables, clear control
flow, and small classes. Preserve extensionless imports unless the project already requires another style. Avoid positional handler
generics, endpoint-specific request types, speculative abstractions, and unrelated restructuring.

## Repository Discovery

Inspect the parts relevant to the request:

- manifests, lockfiles, installed types, scripts, and tests
- app bootstrap, environment loading, router mounting, and route prefixes
- controller conventions, middleware order, and shared status/error responses
- schemas and whether parsed outputs reach application logic
- authentication, groups/scopes, and resource-ownership checks
- services, repositories, shared context, dependency composition, and pagination
- established comments, JSDoc, and spacing

State the approach briefly: **Existing pattern mode** extends local code; **Pattern generation mode** fills a missing pattern. Preserve
compatible conventions, including controller-local error mapping and existing `dependencies/` modules. Fix incompatible behavior within the
requested scope. Do not automatically migrate Express versions, module systems, or deployment platforms.

## Implementation

1. Identify method, path, inputs, authorization, success/error responses, and required dependency access. Resolve material contract gaps
   before affected work; preserve established status codes, headers, payloads, and pagination.
2. Extend the existing router and app mount. Keep prefixes consistent so `/v1` plus `/items` produces `/v1/items` exactly once.
3. Validate body, path, and query before controller execution. For new patterns, store parsed output in `response.locals.validated` and read
   it through local DTO annotations. A schema check does not mutate the original request; consume transformations, coercions, and defaults.
   Preserve rejection-only validation when no downstream behavior relies on discarded output.
4. Authenticate protected routes and enforce groups/scopes and ownership where required. Use one shared `Request` extension for middleware-
   added identity; keep its fields optional until checked. Controllers that do not use identity keep ordinary handler declarations.
5. Keep orchestration in controllers, domain workflows/integrations in services, and datastore requests, transactions, mapping, and cursor
   handling in repositories. Reuse shared clients and repository context through `app.ts` or existing dependency modules. Add only layers
   required by the flow or repeated code; a small project need not have every layer.
6. Reuse error/status utilities. For new patterns, let global error middleware translate typed errors; retain established local `try/catch`
   mappings. Keep parsing, validation, authentication, authorization, and unexpected failures consistent without exposing internal details.
7. Load configuration before dependency initialization. Keep the app importable without starting a listener or loading developer env files;
   process startup owns listening and shutdown. Filenames are examples: preserve an existing `app.ts` entrypoint when appropriate.

Keep HTTP authorization middleware out of scheduled/background workers. Workers call application services with their own configuration.
Database modeling is outside this skill; preserve required atomicity and recovery behavior at repository/service boundaries.

## Express 5 Baseline

Follow supported project dependencies first. For a new project, use Node.js 24, Express 5, TypeScript, and Zod 4 without pinning example patch
versions. Express 5 requires Node.js 18 or higher; verify compatibility before using newer Node APIs.

```bash
npm init
npm install express@5
npm install --save-dev typescript @types/express@5 @types/node
```

Use the project's package manager. Add only middleware and development tools actually used, including their types when needed.

- Rejected promises from async middleware/handlers reach error middleware automatically. Callback-based failures still need `next(error)`.
- Error middleware must declare all four parameters and delegate when headers were already sent.
- `express.json()` and `express.urlencoded()` provide body parsing; `req.body` may be undefined before parsing.
- `req.query` is a getter in Express 5; do not use it as the destination for coerced values.
- Named wildcards (`/*splat`, `/{*splat}`) and braces for optional segments (`/:file{.:ext}`) replace older route syntax.
- Use shared status constants, `Location` for created resources, and bodyless 204 responses.
- Preserve existing auth. For a new Cognito flow, use the connected verification and group middleware example.
- Use `helmet` and the existing CORS policy; new examples use `CORS_ORIGIN`. CORS does not authorize requests. Set proxy trust only for the
  actual trusted topology.

See the official [installation](https://expressjs.com/en/5x/starter/installing/),
[migration](https://expressjs.com/en/guide/migrating-5/), and [error handling](https://expressjs.com/en/guide/error-handling/) guides.
These examples use a TypeScript runner or compilation; native Node type stripping does not type-check or support enum/parameter-property
syntax. Keep the project's existing compiler settings.
When no configuration exists, use the current TypeScript-recommended Node.js module mode (currently `NodeNext`), compatible with the
deployed runtime. Verify current guidance instead of copying an older version-specific fallback; see the app reference for the example.

## Project Shape

Use this layout only when the project lacks an established structure:

```text
src/
├── app.ts                # middleware, routes, and application composition
├── server.ts             # listening and shutdown, when separate startup is useful
├── controllers/
├── data/
│   ├── context.ts
│   └── repositories/
├── dependencies/
├── middlewares/
├── models/
├── routes/
├── services/
└── utilities/
```

Wire only the pieces needed: model → route → controller → service/repository → composition → app mount. Keep database clients out of
controllers; use a shared repository context when repositories work together.

## Environment and Startup

Preserve existing configuration loading. For new projects, commit `.env.example` with all required keys and fictional/empty values;
gitignore local `.env.development` and any optional `.env.production`. Production receives injected values.

Prefer `tsx` for development in new TypeScript projects; preserve an existing working runner. Load local configuration before app imports:

```bash
tsx watch --env-file=.env.development src/server.ts
```

`tsx` runs without type checking; run `tsc --noEmit` separately and use compiled JavaScript for the production examples.
If an existing bootstrap uses `process.loadEnvFile()`, keep it before dependency initialization. Do not duplicate env loading. See
[app-pattern.md](references/app-pattern.md) for startup, scripts, environment examples, and a compatible build configuration.

## References

Load only what the task needs. Preserve the connected code snippets when adapting these patterns:

- [app-pattern.md](references/app-pattern.md): app composition, startup, and environment loading.
- [route-pattern.md](references/route-pattern.md): router paths, authorization, validation, and mounts.
- [controller-pattern.md](references/controller-pattern.md): CRUD handlers, local error mapping, and shared request identity.
- [middleware-pattern.md](references/middleware-pattern.md): authentication, parsed validation output, and global errors.
- [model-pattern.md](references/model-pattern.md): entities, DTOs, and schemas.
- [error-pattern.md](references/error-pattern.md): shared errors and `ErrorResponseBuilder`.
- [status-code-pattern.md](references/status-code-pattern.md): HTTP status constants.
- [pagination-pattern.md](references/pagination-pattern.md): pagination contracts and validated cursors.
- [repository-pattern.md](references/repository-pattern.md): concrete repository classes and shared context/composition.
- [testing-pattern.md](references/testing-pattern.md): isolated HTTP, middleware, and persistence tests.
- [storage-cleanup-pattern.md](references/storage-cleanup-pattern.md): recoverable cross-system deletion when uploads/exports need it.
- [lambda-web-functions.md](references/lambda-web-functions.md): optional deployment without containers.
- [api-review.md](references/api-review.md): completion checklist.

## Validate and Deliver

Read the API review checklist. Run relevant tests, type checks, and build checks. Exercise real middleware through HTTP requests, including
rejected inputs/auth, parsed output, and response contracts. Report unrun checks as unverified; local tests do not prove deployed auth or
external integrations. Do not deploy as validation.

Under `stack-spec-workflow`, follow its active spec and approved Delivery Plan section (or legacy active plan). Record tasks, checks, and
blockers there without competing plans or approval gates.

Finish with implemented routes, meaningful configuration/contract changes, checks and results, and remaining limitations. Include useful
snippets; when implementation is requested, complete the change rather than stopping at a proposal.
