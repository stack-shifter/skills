# JWT Pattern

Use this reference when JWT verification needs to happen inside Lambda code and the repository needs a reusable verification pattern.

## Goal

Verify JWTs in Lambda code using `aws-jwt-verify` instead of ad hoc parsing or a generic JWT library by default.

Do not use this pattern when API Gateway Cognito authorizers already handle route protection unless the user explicitly wants an additional in-Lambda verification step.

## Defaults

- prefer `CognitoJwtVerifier` for Cognito user pool tokens
- prefer `JwtVerifier` for non-Cognito OIDC issuers
- validate issuer, audience, and token use
- treat `decode` as inspection only, not as verification
- prefer API Gateway Cognito authorizers over in-Lambda verification when that option already exists in the repository

## Cognito Example

```ts
import { CognitoJwtVerifier } from 'aws-jwt-verify';

const verifier = CognitoJwtVerifier.create({
    userPoolId: process.env.COGNITO_USER_POOL_ID!,
    tokenUse: 'access',
    clientId: process.env.COGNITO_CLIENT_ID!,
});

const payload = await verifier.verify(token);
```

## Generic OIDC Example

```ts
import { JwtVerifier } from 'aws-jwt-verify';

const verifier = JwtVerifier.create({
    issuer: process.env.JWT_ISSUER!,
    audience: process.env.JWT_AUDIENCE!,
});

const payload = await verifier.verify(token);
```

## Guidance

- Wrap verifier creation in a small service if multiple handlers need token validation
- Prefer one shared helper over repeated verifier setup in each handler
- If the repo already has a token service based on `aws-jwt-verify`, use it instead of this fallback
- If the route already uses a Cognito authorizer at API Gateway, do not add this fallback unless there is a concrete need for in-handler verification
