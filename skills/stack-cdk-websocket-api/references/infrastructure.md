# Infrastructure

The Storm examples apply when the library is installed. For existing project constructs without Storm, use `general-cdk.md`; keep the same application layers.

Use the installed Storm exports. This stack excerpt assumes `props.deploymentStage`, `table`, and validated token settings come from project configuration. A connection table is illustrative; adapt persistence to the project.

## Handler Registry

```ts
// src/handlers/socket.handlers.ts
export const SOCKET_HANDLER = {
    AUTHORIZE: 'authorizeHandler',
    CONNECT: 'connectHandler',
    DISCONNECT: 'disconnectHandler',
    DEFAULT: 'defaultHandler',
    UPDATES: 'updatesHandler',
} as const;
```

Verify each name against the actual entry-module export. Constants do not prove exports exist. Keep this registry free of imports that initialize runtime clients during synthesis.

## Stack Example

```ts
import * as path from 'node:path';
import { WebSocketApi, WEBSOCKET_IDENTITY_SOURCE } from '@stack-shifter/storm-stack';
import { SOCKET_HANDLER } from '../src/handlers/socket.handlers';

const handlerEntry = path.join(__dirname, '../src/handlers/socket.handler.ts');
const functionName = (name: string) => `Updates${name}${props.deploymentStage}`;

const socket = new WebSocketApi(this, 'UpdatesSocket', {
    name: `Updates${props.deploymentStage}`,
    stageName: props.deploymentStage.toLowerCase(),
    routeSelectionExpression: '$request.body.action',
    defaultRouteOptions: {
        environment: { CONNECTION_TABLE: table.tableName },
    },
});

const authorizer = socket.createRequestAuthorizer({
    functionName: functionName('Authorize'),
    entry: path.join(__dirname, '../src/handlers/authorizer.handler.ts'),
    handler: SOCKET_HANDLER.AUTHORIZE,
    identitySource: [WEBSOCKET_IDENTITY_SOURCE.QUERY_STRING],
    environment: { TOKEN_ISSUER: tokenIssuer, TOKEN_AUDIENCE: tokenAudience },
});

socket.addConnectRoute({
    functionName: functionName('Connect'),
    entry: handlerEntry,
    handler: SOCKET_HANDLER.CONNECT,
    authorizer,
    grants: [(fn) => { table.grantWriteData(fn); }],
});

socket.addDisconnectRoute({
    functionName: functionName('Disconnect'),
    entry: handlerEntry,
    handler: SOCKET_HANDLER.DISCONNECT,
    grants: [(fn) => { table.grantWriteData(fn); }],
});

socket.addDefaultRoute({
    functionName: functionName('Default'),
    entry: handlerEntry,
    handler: SOCKET_HANDLER.DEFAULT,
    returnResponse: true,
});

socket.addCustomRoute({
    routeKey: 'updates',
    functionName: functionName('Publish'),
    entry: handlerEntry,
    handler: SOCKET_HANDLER.UPDATES,
    returnResponse: true,
    environment: { MANAGEMENT_API_ENDPOINT: socket.managementApiUrl },
    grantManageConnectionPermission: true,
    grants: [(fn) => { table.grantReadWriteData(fn); }],
});
```

The sender needs read access for recipient lookup and write access for stale-connection deletion. Resource-specific grants can be narrower when the repository supports them. Token verification gets only its own required settings; it does not inherit route defaults.

## Defaults and URLs

- Client connections use `socket.wssApiUrl` (`apiUrl` is an alias). AWS callbacks use `socket.managementApiUrl`, an HTTPS endpoint. Do not derive the callback endpoint from a client custom-domain URL.
- `stageName` defaults to `default`; `autoDeploy` defaults to true; selection defaults to `$request.body.action`. Preserve existing stage names and message routing unless changes are requested.
- Route environment and bundling maps merge shallowly, with route values winning. Grants concatenate. `inheritDefaults: false` skips all route defaults; supply the needed configuration explicitly.
- `grantManageConnectionPermission` grants access for the generated stage. Keep it on send/manage routes rather than in all-route defaults.
- Authentication is attached to the connect route, not to every message route. Authorizer Lambdas are configured separately.
- Routes return generated Lambdas for additional configuration. Use Storm `LambdaNode` directly for related standalone Lambdas rather than creating another wrapper.
- Optional throttling, access logs, detailed metrics, domain mapping, and monitoring belong in API configuration when required. `socket.stage` exposes the generated stage.
