# Middleware Pattern

Use this reference when adding or changing Middy middleware in a DynamoDB-backed API.

Read these files first when they exist:

- `src/middlewares/`
- affected handler files under `src/handlers/`
- `src/utilities/rest-result.ts` or equivalent response helper

## Goal

Centralize request normalization, authorization, validation, and middleware error handling so handlers remain declarative.

## Recommended Middleware Set

- header and event normalization
- JSON body parsing for write routes
- authorization middleware for Cognito groups or JWT claims
- Zod-backed validation middleware for body, query, path, and headers
- shared HTTP error middleware that converts parser failures into standardized responses

## Baseline Handler Chain

```ts
export const createUserHandler = middy(createUserController)
    .use(httpHeaderNormalizer())
    .use(httpEventNormalizer())
    .use(handleHttpError())
    .use(httpJsonBodyParser({ disableContentTypeError: true }))
    .use(authorizedGroup({ allowed: [adminGroup] }))
    .use(validateHeaders(jsonContentTypeSchema))
    .use(validateBody(createUserSchema));
```

## Guidance

- Keep request validation and auth checks in middleware rather than controllers.
- Normalize authorizer claims once so controllers can rely on a stable shape.
- Return shared response-helper objects for expected client errors.
- Log unexpected middleware failures through the shared logger.
- When the repository already has middleware helpers, extend them instead of reimplementing them.
