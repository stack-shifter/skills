# Middleware Pattern

Use this reference when adding or changing reusable Middy middleware in a Postgres-backed API.

## Goal

Keep handlers declarative by centralizing request normalization, authorization, validation, and middleware error translation.

## Middleware Responsibilities

- `injectLambdaContext()` — adapter that bridges `@aws-lambda-powertools/logger/middleware` with Middy v7's handler type; inject after normalizers so the Powertools logger receives the enriched event
- `authorization.middleware.ts` normalizes Cognito claims and enforces allowed groups
- `validation.middleware.ts` validates headers, path params, query params, and body with Zod
- `http-error.middleware.ts` converts parser and middleware exceptions into `RestResult` responses

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
    .use(validateBody(entityPostDtoSchema));
```

## Guidance

- If the repository already has middleware helpers, extend them.
- If it does not, generate middleware as shared reusable units instead of embedding auth or validation in handlers.
- Keep middleware pure and reusable across routes.
- Put request-shape validation in middleware, not controllers.
- Normalize claims once and pass normalized values through `requestContext.authorizer`.
- Return `RestResult` values for expected request failures such as bad input or missing auth context.
- Log unexpected middleware failures through the shared logger before returning a generic 500 response.
