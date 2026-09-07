---
name: stack-cdk-websocket-api
description: Build AWS API Gateway WebSocket APIs with Storm when installed or Storm-style project-local constructs, including connection authentication, message routes, connection tracking, and callbacks. Use for WebSocket infrastructure and its handlers, middleware, controllers, services, and repositories; use the REST skill for HTTP endpoints.
---

## Purpose

Choose infrastructure from installed dependencies and preserve the project's familiar application structure for behavior. Keep examples connected: authenticate a connection, store trusted identity, validate an update message, notify authorized recipients, and clean up connections.

## Discover First

Inspect package manifests, installed Storm exports and types, stack routes, handler registries, middleware, controllers, services, repositories, composition modules, and tests. Follow explicit requirements, then existing application contracts, then these examples.

- **Existing pattern mode:** extend existing route wiring and the existing application layers.
- **Pattern generation mode:** use installed Storm or Storm-style project-local constructs and the connected examples below for missing pieces. Create only the application modules needed by the flow.

Prefer Storm's supported constructs when `@stack-shifter/storm-stack` is installed. Otherwise extend existing project constructs or build Storm-style project-local constructs around native CDK resources. Explicit user preferences take precedence. Do not automatically install Storm or migrate existing infrastructure; adding a route should not replace its API. Inspect the actual exports and signatures before choosing examples. Keep native resource wiring inside API/Lambda constructs and stack modules focused on composition. Implement only the construct capabilities needed by the requested flow; preserve compatible existing interfaces.

## Implementation Steps

1. Identify connection authentication, route keys, message shapes, recipient authorization, acknowledgments, and expiry behavior. Preserve existing contracts; resolve material gaps before affected work.
2. Register routes with installed Storm or project-local WebSocket constructs. Keep handler names centralized, check actual exports, and give each Lambda only its required resource access.
3. Authenticate at `$connect` and save trusted identity with the connection. For later messages, load trusted connection state and enforce current authorization rather than accepting identity from message bodies.
4. Keep handlers declarative with `withCommonMiddleware` and `withMessageMiddleware`. Parse and validate messages before controllers run; translate expected failures consistently.
5. Put connection persistence behind repositories and callbacks behind a messaging service. Compose reusable clients, context, and services in `app.ts` or existing dependency modules.
6. Handle disconnects idempotently, filter expired connections, and remove stale connections reported by callbacks. Define partial-delivery behavior without promising guaranteed delivery.
7. Validate the change before reporting completion. Do not deploy merely to validate examples.

Prefer arrow functions for standalone functions, handlers, and middleware callbacks in generated examples. Keep ordinary constructors and class methods for the existing service and repository structure.

## Project Shape

```text
lib/
├── constructs/              # project configuration helpers, not copied Storm constructs
└── websocket-stack.ts
src/
├── app.ts
├── controllers/socket.controller.ts
├── data/
│   ├── context.ts
│   └── repositories/connection.repository.ts
├── handlers/
│   ├── socket.handlers.ts   # dependency-free registry
│   ├── socket.handler.ts
│   └── authorizer.handler.ts
├── middlewares/socket.middleware.ts
├── models/socket.model.ts
├── services/
│   ├── token.service.ts
│   └── messaging.service.ts
└── utilities/
```

Preserve an established `dependencies/` composition root instead of introducing a competing one. Handlers compose middleware; controllers orchestrate; services handle integrations; repositories own datastore requests. Database modeling and frontend implementation are separate tasks.

## References

Load only the relevant references; the snippets are related application modules, not a standalone generated project.

- `references/general-cdk.md`: project-local WebSocket/Lambda constructs and application helper imports when Storm is absent.
- `references/infrastructure.md`: direct Storm constructs, route registry, defaults, grants, and URLs.
- `references/authentication.md`: request authorizer, trusted identity, and action authorization.
- `references/middleware-messages.md`: typed common/message middleware, validation, and response semantics.
- `references/connection-lifecycle.md`: repository contract and connect/disconnect/update controllers.
- `references/runtime-messaging.md`: repository context, `app.ts`, and callback service.
- `references/review.md`: verification and representative scenarios.

## Validate and Deliver

Read `references/review.md`. Run relevant tests, type checks, and affected CDK synthesis when tooling is available. Report unavailable checks as unverified, with the reason. A static review does not verify deployed behavior.

When used with `stack-spec-workflow`, follow the active delivery spec and approved plan, recording completed tasks, validation, and blockers there. Do not add competing plans or approval gates.

Finish with routes implemented, client and callback configuration, message-contract changes, checks, and material limitations. Include snippets only when useful; complete requested implementation rather than stopping at examples.
