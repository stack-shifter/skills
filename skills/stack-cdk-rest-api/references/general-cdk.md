# Project-Local REST Constructs

When Storm is absent, reuse compatible project constructs or build small local constructs with the same composition style. Native CDK resource wiring belongs inside these constructs. Preserve existing interfaces rather than forcing a rename or migration.

These examples implement only the required route composition, shared environment, authorization, and explicit grants. They do not promise full Storm compatibility. Add request models, logging, CORS, or other options only when required.

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

## Local REST Construct

```ts
// lib/constructs/rest-api.ts
import { Construct } from 'constructs';
import * as gateway from 'aws-cdk-lib/aws-apigateway';
import type { IFunction } from 'aws-cdk-lib/aws-lambda';
import { LambdaNode, LambdaNodeProps } from './lambda-node';

type Grant = (fn: IFunction) => void;
type RouteDefaults = {
    environment?: Record<string, string>;
    authorizer?: gateway.IAuthorizer | null;
    scopes?: string[];
};
type RouteOptions = LambdaNodeProps & RouteDefaults & {
    routePath: string;
    grants?: Grant[];
};

export class RestApi extends Construct {
    readonly api: gateway.RestApi;
    private readonly defaults: RouteDefaults;

    constructor(scope: Construct, id: string, props: {
        name: string;
        stageName: string;
        defaultRouteOptions?: RouteDefaults;
    }) {
        super(scope, id);
        this.defaults = props.defaultRouteOptions ?? {};
        this.api = new gateway.RestApi(this, 'Api', {
            restApiName: props.name,
            deployOptions: { stageName: props.stageName },
        });
    }

    get(options: RouteOptions) { return this.addRoute('GET', options); }
    post(options: RouteOptions) { return this.addRoute('POST', options); }
    put(options: RouteOptions) { return this.addRoute('PUT', options); }
    delete(options: RouteOptions) { return this.addRoute('DELETE', options); }

    private addRoute(method: string, options: RouteOptions) {
        if (!options.routePath.startsWith('/')) throw new Error('Route path must start with /');
        const { routePath, grants = [], authorizer, scopes, ...lambdaProps } = options;
        const routeAuthorizer = authorizer === undefined ? this.defaults.authorizer : authorizer;
        const fn = new LambdaNode(this, `${method}-${routePath}`, {
            ...lambdaProps,
            environment: { ...this.defaults.environment, ...lambdaProps.environment },
        }).function;
        for (const grant of grants) grant(fn);

        let resource = this.api.root;
        for (const segment of routePath.split('/').filter(Boolean)) {
            resource = resource.getResource(segment) ?? resource.addResource(segment);
        }
        resource.addMethod(method, new gateway.LambdaIntegration(fn), {
            authorizer: routeAuthorizer ?? undefined,
            authorizationType: routeAuthorizer ? gateway.AuthorizationType.COGNITO : gateway.AuthorizationType.NONE,
            authorizationScopes: routeAuthorizer ? (scopes ?? this.defaults.scopes) : undefined,
        });
        return fn;
    }
}
```

This baseline supports Cognito authorizers. Add explicit support for other authorization types if needed; do not pass a custom Lambda authorizer while retaining `COGNITO`. Omitted route auth inherits the default; `null` makes a route public. Scope arrays replace defaults. Grants are route-specific; environment maps merge with route values winning.

## Stack Composition

The stack supplies `props`, `userPool`, `table`, and `readScope`. Its path and naming helpers remain straightforward arrow functions.

```ts
import * as path from 'node:path';
import { CognitoUserPoolsAuthorizer } from 'aws-cdk-lib/aws-apigateway';
import { RestApi } from './constructs/rest-api';
import { CLIENT_HANDLER } from '../src/handlers/client.handler';

const authorizer = new CognitoUserPoolsAuthorizer(this, 'ClientsAuthorizer', {
    cognitoUserPools: [userPool],
});
const api = new RestApi(this, 'ClientIntake', {
    name: `ClientIntake-${props.deploymentStage}`,
    stageName: props.deploymentStage.toLowerCase(),
    defaultRouteOptions: { authorizer, environment: { TABLE_NAME: table.tableName } },
});
api.get({
    routePath: '/clients',
    entry: path.join(__dirname, '../src/handlers/client.handler.ts'),
    handler: CLIENT_HANDLER.QUERY,
    scopes: [readScope],
    grants: [(fn) => { table.grantReadData(fn); }],
});
```

Keep the same middleware, controllers, repositories, context, `app.ts`, and response helpers on either infrastructure path. Wrap recurring API or scheduled-job wiring in the project's constructs when needed; do not spread low-level integrations through stack modules.
