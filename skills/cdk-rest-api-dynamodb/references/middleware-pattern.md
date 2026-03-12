# Middleware Pattern

Use this reference when adding or changing reusable Middy middleware in a DynamoDB-backed API.

## Goal

Centralize request normalization, authorization, validation, and middleware error handling so handlers remain declarative.

## Recommended Middleware Set

- `httpHeaderNormalizer()` — normalizes header casing before any middleware reads them
- `httpEventNormalizer()` — normalizes the API Gateway event shape for consistent access
- `injectLambdaContext()` — adapter that bridges `@aws-lambda-powertools/logger/middleware` with Middy v7's handler type; inject after normalizers so the Powertools logger receives the enriched event
- `handleHttpError()` — converts Middy parser errors (415, 422) into standardized `RestResult` responses
- `httpJsonBodyParser({ disableContentTypeError: true })` — parses the JSON body for write routes; `disableContentTypeError` defers content-type enforcement to the Zod header schema
- `authorizedGroup(...)` — Cognito group-based authorization middleware
- `validateHeaders(...)`, `validatePathParameters(...)`, `validateQueryParameters(...)`, `validateBody(...)` — Zod-backed validation middleware

## Handler Composition Pattern

Use `withCommonMiddleware` and `withWriteMiddleware` factory helpers inside each handler file rather than repeating the base chain inline. This is the established pattern across all handler files.

```ts
import middy from '@middy/core';
import httpEventNormalizer from '@middy/http-event-normalizer';
import httpHeaderNormalizer from '@middy/http-header-normalizer';
import httpJsonBodyParser from '@middy/http-json-body-parser';
import { injectLambdaContext } from '../middlewares/inject-lambda-context.middleware';
import { handleHttpError } from '../middlewares/http-error.middleware';
import { logger } from '../services/logger';

/**
 * Base Middy pipeline shared by every route in this handler file.
 * Normalizes headers and event shape, injects Powertools logger context,
 * and maps parser errors to standard responses.
 */
const withCommonMiddleware = <T extends typeof getByIdController>(handler: T) =>
    middy(handler)
        .use(httpHeaderNormalizer())
        .use(httpEventNormalizer())
        .use(injectLambdaContext(logger, { logEvent: false }))
        .use(handleHttpError());

/**
 * Extended pipeline for write routes (POST, PUT).
 * Adds JSON body parsing on top of the common middleware.
 * `disableContentTypeError` defers content-type enforcement to the Zod header schema.
 */
const withWriteMiddleware = <T extends typeof saveController>(handler: T) =>
    withCommonMiddleware(handler).use(httpJsonBodyParser({ disableContentTypeError: true }));
```

Append route-specific middleware (auth, validation) to the returned instance:

```ts
export const getByIdEntityHandler = withCommonMiddleware(getByIdEntityController)
    .use(authorizedGroup({ allowed: [allowedGroup] }))
    .use(validatePathParameters(entityPathSchema));

export const saveEntityHandler = withWriteMiddleware(saveEntityController)
    .use(authorizedGroup({ allowed: [allowedGroup] }))
    .use(validateHeaders(jsonContentTypeSchema))
    .use(validatePathParameters(organizationPathSchema))
    .use(validateBody(entityPostDtoSchema));
```

## Guidance

- Keep request validation and auth checks in middleware rather than controllers.
- Normalize authorizer claims once so controllers can rely on a stable shape.
- Return shared response-helper objects for expected client errors.
- Log unexpected middleware failures through the shared logger.
- When the repository already has middleware helpers, extend them instead of reimplementing them.
- When it does not, generate middleware as shared reusable units instead of embedding checks directly in handlers.
