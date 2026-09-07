# Auth Pattern

Use this reference when a change affects protected routes, Cognito scopes, in-handler authorization behavior, or the exceptional case where JWT verification happens inside Lambda.

## Goal

Preserve or generate a coherent authorization model across the stack layer and handler layer.

## Common Shape

A strong baseline protects routes through:

1. API Gateway Cognito authorizer plus route scopes in CDK
2. handler-level group checks through `authorizedGroup(...)`

## Baseline Route Example

```ts
import { PROJECT_HANDLER } from "../src/handlers/project.handler";

api.setDefaultRouteOptions({
    authorizer: api.createCognitoAuthorizer(userPool),
    environmentVariables: getDefaultLambdaEnvironment(),
});

api.post({
    routePath: "/projects",
    lambdaName: lambdaName("ProjectsPost"),
    filePath: handlerPath("src/handlers/project.handler.ts"),
    handlerName: PROJECT_HANDLER.SAVE,
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
- Use existing `ALLOWED_GROUP` or equivalent group semantics when present.
- If the repository lacks handler-level authorization middleware but needs it, generate one reusable middleware instead of inlining claim checks.
- Prefer API Gateway Cognito authorizers over ad hoc in-Lambda JWT verification.
- If in-Lambda JWT verification is explicitly required, treat it as an exception and keep it consistent with the existing auth flow instead of replacing the Gateway authorizer.

## In-Lambda JWT Verification Exception

Only use this pattern when JWT verification must happen inside Lambda code.

Defaults:

- prefer `CognitoJwtVerifier` for Cognito user pool tokens
- prefer `JwtVerifier` for non-Cognito OIDC issuers
- validate issuer, audience, and token use
- treat `decode` as inspection only, not as verification

```ts
import { CognitoJwtVerifier } from 'aws-jwt-verify';

const verifier = CognitoJwtVerifier.create({
    userPoolId: process.env.COGNITO_USER_POOL_ID!,
    tokenUse: 'access',
    clientId: process.env.COGNITO_CLIENT_ID!,
});

const payload = await verifier.verify(token);
```
