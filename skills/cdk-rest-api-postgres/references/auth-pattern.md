# Auth Pattern

Use this reference when a change affects protected routes, Cognito scopes, or in-handler authorization behavior.

## Goal

Preserve or generate a coherent two-layer authorization model.

## Common Shape

A strong baseline protects routes through:

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

- If the repository already has route scopes in the stack layer, preserve them.
- If it already has `authorizedGroup(...)` or equivalent middleware, preserve or extend it when group membership matters.
- Use existing `ADMIN_GROUP` or equivalent group semantics when present.
- If the repository lacks handler-level authorization middleware but needs it, generate one reusable middleware instead of inlining claim checks.
- Prefer API Gateway Cognito authorizers over ad hoc in-Lambda JWT verification.
- If in-Lambda JWT verification is explicitly required, add it as an exception and keep it consistent with the existing auth flow instead of replacing the Gateway authorizer.
