# Project-Local WebSocket Constructs

When Storm is absent, reuse compatible project constructs or build small local constructs. Keep native integration, stage, and Lambda wiring inside them; stacks register routes using readable methods. These examples implement a focused subset, not a copy of the Storm library.

## Local Lambda Construct

```ts
// lib/constructs/lambda-node.ts
import { Construct } from 'constructs';
import { Duration } from 'aws-cdk-lib';
import { NodejsFunction, NodejsFunctionProps } from 'aws-cdk-lib/aws-lambda-nodejs';

export interface LambdaNodeProps extends NodejsFunctionProps {
    entry: string;
    handler: string;
}

export class LambdaNode extends Construct {
    readonly function: NodejsFunction;

    constructor(scope: Construct, id: string, props: LambdaNodeProps) {
        super(scope, id);
        this.function = new NodejsFunction(this, 'Function', {
            timeout: Duration.seconds(15),
            ...props,
        });
    }
}
```

Keep runtime, architecture, and bundling defaults aligned with the project's supported dependencies. Reuse existing naming and environment helpers; they remain application-owned.

## Local WebSocket Construct

```ts
// lib/constructs/websocket-api.ts
import { Construct } from 'constructs';
import * as gateway from 'aws-cdk-lib/aws-apigatewayv2';
import { WebSocketLambdaIntegration } from 'aws-cdk-lib/aws-apigatewayv2-integrations';
import { WebSocketLambdaAuthorizer } from 'aws-cdk-lib/aws-apigatewayv2-authorizers';
import type { IFunction } from 'aws-cdk-lib/aws-lambda';
import { LambdaNode, LambdaNodeProps } from './lambda-node';

type RouteOptions = LambdaNodeProps & {
    grants?: Array<(fn: IFunction) => void>;
    returnResponse?: boolean;
    grantManageConnectionPermission?: boolean;
};

export class WebSocketApi extends Construct {
    readonly api: gateway.WebSocketApi;
    readonly stage: gateway.WebSocketStage;
    readonly wssApiUrl: string;
    readonly managementApiUrl: string;
    private readonly environment: Record<string, string>;

    constructor(scope: Construct, id: string, props: {
        name: string;
        stageName: string;
        defaultRouteOptions?: { environment?: Record<string, string> };
    }) {
        super(scope, id);
        this.environment = props.defaultRouteOptions?.environment ?? {};
        this.api = new gateway.WebSocketApi(this, 'Api', {
            apiName: props.name,
            routeSelectionExpression: '$request.body.action',
        });
        this.stage = new gateway.WebSocketStage(this, 'Stage', {
            webSocketApi: this.api, stageName: props.stageName, autoDeploy: true,
        });
        this.wssApiUrl = this.stage.url;
        this.managementApiUrl = this.stage.callbackUrl;
    }

    createRequestAuthorizer(props: LambdaNodeProps & { identitySource: string[] }) {
        const { identitySource, ...lambdaProps } = props;
        const fn = new LambdaNode(this, 'AuthorizerFunction', lambdaProps).function;
        return new WebSocketLambdaAuthorizer('Authorizer', fn, { identitySource });
    }

    addConnectRoute(props: RouteOptions & { authorizer?: WebSocketLambdaAuthorizer }) {
        const { authorizer, ...options } = props;
        return this.addRoute('$connect', options, authorizer);
    }
    addDisconnectRoute(props: RouteOptions) { return this.addRoute('$disconnect', props); }
    addDefaultRoute(props: RouteOptions) { return this.addRoute('$default', props); }
    addCustomRoute(props: RouteOptions & { routeKey: string }) {
        const { routeKey, ...options } = props;
        return this.addRoute(routeKey, options);
    }

    private addRoute(key: string, props: RouteOptions, authorizer?: WebSocketLambdaAuthorizer) {
        const { grants = [], returnResponse, grantManageConnectionPermission, ...lambdaProps } = props;
        const fn = new LambdaNode(this, `Route-${key}`, {
            ...lambdaProps,
            environment: { ...this.environment, ...lambdaProps.environment },
        }).function;
        for (const grant of grants) grant(fn);
        if (grantManageConnectionPermission) this.stage.grantManagementApiAccess(fn);
        this.api.addRoute(key, {
            integration: new WebSocketLambdaIntegration(key, fn),
            authorizer, returnResponse,
        });
        return fn;
    }
}
```

The baseline shares environment only; route grants remain explicit, and the authorizer does not inherit route settings. Expand default options, domain mapping, and monitoring only when required, documenting merge behavior. Clients use `wssApiUrl`; senders use `managementApiUrl`.

## Stack Composition

The stack supplies the table, stage, and validated token configuration. Application handler exports match `infrastructure.md`.

```ts
import * as path from 'node:path';
import { WebSocketApi } from './constructs/websocket-api';
import { SOCKET_HANDLER } from '../src/handlers/socket.handlers';

const entry = path.join(__dirname, '../src/handlers/socket.handler.ts');
const socket = new WebSocketApi(this, 'UpdatesSocket', {
    name: `Updates${props.deploymentStage}`,
    stageName: props.deploymentStage.toLowerCase(),
    defaultRouteOptions: { environment: { CONNECTION_TABLE: table.tableName } },
});
const authorizer = socket.createRequestAuthorizer({
    entry: path.join(__dirname, '../src/handlers/authorizer.handler.ts'),
    handler: SOCKET_HANDLER.AUTHORIZE,
    identitySource: ['route.request.querystring.authorization'],
    environment: { TOKEN_ISSUER: tokenIssuer, TOKEN_AUDIENCE: tokenAudience },
});
socket.addConnectRoute({
    entry, handler: SOCKET_HANDLER.CONNECT, authorizer,
    grants: [(fn) => { table.grantWriteData(fn); }],
});
socket.addDisconnectRoute({
    entry, handler: SOCKET_HANDLER.DISCONNECT,
    grants: [(fn) => { table.grantWriteData(fn); }],
});
socket.addDefaultRoute({ entry, handler: SOCKET_HANDLER.DEFAULT, returnResponse: true });
socket.addCustomRoute({
    routeKey: 'updates', entry, handler: SOCKET_HANDLER.UPDATES, returnResponse: true,
    environment: { MANAGEMENT_API_ENDPOINT: socket.managementApiUrl },
    grantManageConnectionPermission: true,
    grants: [(fn) => { table.grantReadWriteData(fn); }],
});
```

## Application Helpers Without Storm

In the other references, replace imports of `SocketResult` and `createAuthorizerPolicy` from Storm with the existing application utilities. If absent, implement the small helpers required by those examples. Do not install Storm solely for these helpers.

```ts
// src/utilities/socket-result.ts
import type { APIGatewayProxyStructuredResultV2 } from 'aws-lambda';

const response = (statusCode: number, body?: unknown): APIGatewayProxyStructuredResultV2 => ({
    statusCode,
    body: body === undefined ? '' : JSON.stringify(body),
});

export const SocketResult = {
    ok: (body?: unknown) => response(200, body),
    badRequest: (message: string) => response(400, { message }),
    unauthorized: (message: string) => response(401, { message }),
    forbidden: (message: string) => response(403, { message }),
    internalServerError: (message: string) => response(500, { message }),
};
```

This is an illustrative new application envelope; preserve an existing envelope or intentionally match the library contract when migrating.

```ts
// src/utilities/authorizer.ts
import type { APIGatewayAuthorizerResult, APIGatewayAuthorizerResultContext } from 'aws-lambda';

export const createAuthorizerPolicy = (
    allow: boolean,
    methodArn: string,
    context?: APIGatewayAuthorizerResultContext,
): APIGatewayAuthorizerResult => ({
    principalId: 'socket-authorizer',
    policyDocument: {
        Version: '2012-10-17',
        Statement: [{ Action: 'execute-api:Invoke', Effect: allow ? 'Allow' : 'Deny', Resource: methodArn }],
    },
    context,
});
```

The existing token service still verifies credentials; a policy helper is not a verifier. The middleware and controllers keep their existing organization and arrow-function style.
