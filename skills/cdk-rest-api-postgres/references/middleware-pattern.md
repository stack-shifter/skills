# Middleware Pattern

Use this reference when adding or changing Middy middleware in a Postgres-backed API.

Read these files first when they exist:

- `src/middlewares/authorization.middleware.ts`
- `src/middlewares/validation.middleware.ts`
- `src/middlewares/http-error.middleware.ts`
- affected handler files under `src/handlers/`

## Goal

Keep handlers declarative by centralizing request normalization, authorization, validation, and middleware error translation.

## Middleware Responsibilities

- `authorization.middleware.ts` normalizes Cognito claims and enforces allowed groups
- `validation.middleware.ts` validates headers, path params, query params, and body with Zod
- `http-error.middleware.ts` converts parser and middleware exceptions into `RestResult` responses

## Baseline Handler Chain

```ts
export const saveProjectHandler = middy(saveProjectController)
    .use(httpHeaderNormalizer())
    .use(httpEventNormalizer())
    .use(handleHttpError())
    .use(httpJsonBodyParser({ disableContentTypeError: true }))
    .use(authorizedGroup({ allowed: [adminGroup] }))
    .use(validateHeaders(jsonContentTypeSchema))
    .use(validateBody(projectPostDtoSchema));
```

## Guidance

- Keep middleware pure and reusable across routes.
- Put request-shape validation in middleware, not controllers.
- Normalize claims once and pass normalized values through `requestContext.authorizer`.
- Return `RestResult` values for expected request failures such as bad input or missing auth context.
- Log unexpected middleware failures through the shared logger before returning a generic 500 response.
