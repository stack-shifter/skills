# Auth Pattern

Use this reference when a change affects protected routes, Cognito scopes, or in-handler authorization behavior.

Read these files first:

- `lib/core-stack.ts`
- `src/middlewares/authorization.middleware.ts`
- affected handler files under `src/handlers/`

## Goal

Preserve the repository's two-layer authorization model.

## Repository Shape

This project protects routes through:

1. API Gateway Cognito authorizer plus route scopes in CDK
2. handler-level group checks through `authorizedGroup(...)`

## Baseline Route Example

```ts
api.setDefaultRouteOptions({
    authorizer: api.createCognitoAuthorizer(userPool),
    environmentVariables: getDefaultLambdaEnvironment(),
});

api.post({
    routePath: "/projects",
    lambdaName: lambdaName("ProjectsPost"),
    filePath: handlerPath("src/handlers/project.handler.ts"),
    handlerName: "saveProjectHandler",
    description: "/projects",
    scopes: [writeAuthScope],
});
```

## Baseline Handler Example

```ts
export const saveProjectHandler = withWriteMiddleware(saveProjectController)
    .use(authorizedGroup({ allowed: [adminGroup] }))
    .use(validateHeaders(jsonContentTypeSchema))
    .use(validateBody(projectPostDtoSchema));
```

## Guidance

- Preserve route scopes in `lib/core-stack.ts` when modifying protected endpoints.
- Preserve or extend `authorizedGroup(...)` usage in handlers when group membership matters.
- Use `ADMIN_GROUP` semantics already established in the repository.
- Prefer API Gateway Cognito authorizers over ad hoc in-Lambda JWT verification.
- If in-Lambda JWT verification is explicitly required, add it as an exception and keep it consistent with the existing auth flow instead of replacing the Gateway authorizer.
