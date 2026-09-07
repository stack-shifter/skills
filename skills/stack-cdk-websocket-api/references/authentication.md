# Authentication and Authorization

Use the displayed Storm helper imports when installed. Without Storm, use the application helper imports described in `general-cdk.md`; the application flow stays the same.

Authenticate during the connection handshake. Store a stable subject and server-authorized recipient scope with the connection. A client-provided subject or topic is not authorization.

## Request Authorizer

This example uses an application `tokenService.verify` that validates signature, issuer, audience, expiry, and applicable token use. It returns a subject, recipient scope, and expiry derived from trusted identity and policy. Implement it with the project's supported verifier; do not copy hardcoded issuers or assume these fields exist in every token.

```ts
// src/handlers/authorizer.handler.ts
import type { APIGatewayRequestAuthorizerHandler } from 'aws-lambda';
import { createAuthorizerPolicy } from '@stack-shifter/storm-stack';
import { tokenService } from '../services/token.service';

export const authorizeHandler: APIGatewayRequestAuthorizerHandler = async (event) => {
    const token = event.queryStringParameters?.authorization;
    if (!token) return createAuthorizerPolicy(false, event.methodArn);

    try {
        const identity = await tokenService.verify(token);
        return createAuthorizerPolicy(true, event.methodArn, {
            subjectId: identity.subjectId,
            scopeId: identity.scopeId,
            expiresAt: identity.expiresAt,
        });
    } catch {
        return createAuthorizerPolicy(false, event.methodArn);
    }
};
```

`tokenService` is a module-scope application instance configured from validated `TOKEN_ISSUER` and `TOKEN_AUDIENCE`. It is separate from the connection runtime so the authorizer does not initialize repositories or callback clients.

The query-string identity source is an example matching Storm's default. Choose the transport for actual clients: browser WebSocket constructors cannot set arbitrary Authorization headers. Do not log handshake tokens or full credential-bearing URLs. For an existing handshake, preserve its contract.

## Message Authorization

Connect authorization does not authenticate every later message anew. Load the sender connection from the repository using `requestContext.connectionId`, reject expired or missing state, and check current publish permission before querying recipients. The lifecycle example's `updatePermissions.canPublish` is an application policy service, not a Storm export.

Scope recipient queries using trusted connection information. If permission changes must take effect immediately, perform the current permission check or explicitly revoke connections; token expiry alone is not immediate revocation.

Keep any familiar `authorizedGroup(...)` API in application middleware only when adapted to this trusted WebSocket identity. Do not reuse REST-specific claim paths or depend on connect-authorizer context appearing on later events.
