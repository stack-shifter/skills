# Auth Pattern

Use this reference when a change affects protected routes, Cognito scopes, in-handler authorization behavior, or the exceptional case where JWT verification happens inside Lambda.

## Goal

Preserve or generate a coherent authorization model across the stack layer and handler layer.

## Common Shape

A strong baseline protects routes through:

1. API Gateway Cognito authorizer plus route scopes in CDK
2. handler-level group checks through `authorizedGroup(...)`

## Storm Route Example

Use the Storm API instance and stack helpers from `rest-api-pattern.md`. Its constructor defaults already include the Cognito authorizer and shared environment.

```ts
api.post({
    routePath: '/projects',
    functionName: lambdaName('ProjectsPost'),
    entry: handlerPath('src/handlers/project.handler.ts'),
    handler: PROJECT_HANDLER.SAVE,
    scopes: [writeAuthScope],
});
```

`PROJECT_HANDLER` is the project's handler registry. Keep application authorization middleware separate from Storm’s route configuration.

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

## Custom Token Authorizers

When the API requires a Lambda token authorizer, Storm’s `createTokenAuthorizer(...)` accepts `identitySource`, `validationRegex`, and `resultsCacheTtl` alongside its Lambda configuration and grants. The defaults use the Authorization header, a Bearer-token validation expression, and a 60-second cache.

Use `validationRegex: null` only when intentionally disabling Gateway’s token-format check; the authorizer Lambda must still authenticate the token. Customize the header or expression only for the required authentication contract. Keep `authorizedGroup(...)` and application authorization separate.

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

## Compatibility and Validation

Use the separate Storm route interface in `rest-api-pattern.md` when the library is installed. Preserve `authorizedGroup(...)` and existing group policy. Test missing authentication context, insufficient permission, and successful access; failed authorization must prevent controller execution. Configuring Gateway authentication does not grant a Lambda permission to administer the user pool.
