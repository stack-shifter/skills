# REST API Pattern

Use this reference when you need a reusable pattern for centralized API Gateway REST route composition.

## Goal

Add or modify REST endpoints in a way that stays centralized and repeatable.

## Common Shape

- API Gateway REST API through Storm `RestApi`
- one Lambda handler export per route
- shared Cognito authorizer supplied through constructor route defaults
- per-route Cognito scopes passed from the stack layer
- route files wired through shared path and naming helpers when available

Check installed exports before using project helpers or optional API settings. Storm supports `stageName`; omitting it keeps the `default` stage unless `deployOptions.stageName` is supplied.

## Storm Example

Use this stack excerpt for new APIs. `props`, `userPool`, `table`, and `readScope` come from the stack. The naming/environment helpers and handler registry are project-local; their responsibilities are described in `node-lambda-pattern.md`.

```ts
import { CognitoUserPoolsAuthorizer } from 'aws-cdk-lib/aws-apigateway';
import { RestApi } from '@stack-shifter/storm-stack';
import * as path from 'node:path';
import { CLIENT_HANDLER } from '../src/handlers/client.handler';
import { getDefaultLambdaEnvironment, getStackLambdaName } from './constructs/node-lambda';

const lambdaName = (name: string) => getStackLambdaName(name, props.deploymentStage);
const handlerPath = (filePath: string) => path.join(__dirname, '..', filePath);

const authorizer = new CognitoUserPoolsAuthorizer(this, 'ClientsAuthorizer', {
    cognitoUserPools: [userPool],
});

const api = new RestApi(this, 'ClientIntake', {
    name: `ClientIntake-${props.deploymentStage}`,
    stageName: props.deploymentStage.toLowerCase(),
    corsOrigins: [process.env.CORS_ORIGIN || '*'],
    defaultRouteOptions: {
        authorizer,
        environment: getDefaultLambdaEnvironment(),
    },
});

api.get({
    routePath: '/clients',
    functionName: lambdaName('ClientsQuery'),
    entry: handlerPath('src/handlers/client.handler.ts'),
    handler: CLIENT_HANDLER.QUERY,
    scopes: [readScope],
    grants: [(fn) => { table.grantReadData(fn); }],
});
```

Choose CORS origins from the project's requirements and verify the installed library's exports and signatures.

### Storm Defaults and Overrides

Verify these behaviors against the installed version:

- Route properties override shared defaults. `environment` and `bundling` merge shallowly, with route values winning on matching keys.
- Default and route `grants` concatenate. A route's empty grant array does not remove inherited permissions. Put resource grants on the routes that need them.
- `authorizer: null` explicitly makes a route public. Omit the property to inherit authorization; do not rely on an explicit `undefined` value.
- `inheritDefaults: false` skips all API-level route defaults, including environment, authorization, and grants. Supply every setting the route still requires; Lambda wrapper defaults remain separate.
- Scope arrays replace rather than concatenate. Check the resulting authorizer and scopes together.
- Route methods return the generated Lambda for additional explicit configuration.

## Stage and Operational Settings

- `stageName` takes precedence over `deployOptions.stageName`, then falls back to `default`. The example names a new stage after the deployment environment; preserve an existing API’s stage name unless a change is requested.
- Use `deployOptions` for native REST stage settings. Top-level `throttle`, `detailedMetricsEnabled`, and `accessLogSettings` override their corresponding stage settings when supplied. These controls can be used without enabling execution logging.
- Use `monitoring` for alarms independently of logging. `monitoring: false` disables alarms, including those otherwise created by `logging.enabled`. When omitted, enabled logging retains its existing alarm behavior.
- The generated stage is exposed as `api.stage`. Use it when another resource needs the actual stage rather than reconstructing its name.

Add these options only when the task needs them and the installed Storm version supports them.

## HANDLER Registry Pattern

Preserve the project’s typed handler registry convention. Keep a new shared registry in a dependency-free module when importing the handler would initialize runtime dependencies during synthesis. The registry centralizes names, but a string constant does not prove the corresponding function export exists. Verify each configured name against the entry module.

```ts
export const CLIENT_HANDLER = {
    QUERY: "queryClientHandler",
    GET_BY_ID: "getByIdClientHandler",
    SAVE: "saveClientHandler",
    UPDATE_BY_ID: "updateByIdClientHandler",
    DELETE: "deleteClientHandler",
} as const;
```

Import and use the registry in the stack for all route wiring:

```ts
import { CLIENT_HANDLER } from "../src/handlers/client.handler";

api.get({
    routePath: "/clients",
    functionName: lambdaName("ClientsQuery"),
    entry: handlerPath("src/handlers/client.handler.ts"),
    handler: CLIENT_HANDLER.QUERY,
    description: "/clients",
    scopes: readScopes,
});

api.post({
    routePath: "/clients",
    functionName: lambdaName("ClientsPost"),
    entry: handlerPath("src/handlers/client.handler.ts"),
    handler: CLIENT_HANDLER.SAVE,
    description: "/clients",
    scopes: writeScopes,
});
```

## Guidance

- If the repository already has centralized route registration, extend it.
- If it does not, register routes together using Storm `RestApi`.
- Prefer shared route helpers such as `get`, `post`, `put`, and `delete` over raw `addMethod` once the pattern exists.
- Reuse the `HANDLER` registry and verify the actual export names. Avoid introducing runtime initialization into infrastructure imports.
- `routePath` must start with `/`.
- Shared environment variables belong in one common helper or route-defaults path, not duplicated inline on every route.
- Keep persistence concerns out of route registration. Route registration should focus on handler wiring, auth, scopes, and shared environment defaults.
