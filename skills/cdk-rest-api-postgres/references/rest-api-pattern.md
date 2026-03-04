# REST API Pattern

Use this reference when you need a reusable pattern for centralized API Gateway REST route composition.

## Goal

Add or modify REST endpoints in a way that stays centralized and repeatable.

## Common Shape

- API Gateway REST API via a reusable stack or construct layer
- one Lambda handler export per route
- shared Cognito authorizer set with a default-route helper
- per-route Cognito scopes passed from the stack layer
- route files wired through shared path and naming helpers when available

## Baseline Example

```ts
import * as path from "node:path";
import { RestServerlessApi } from "./constructs/rest-api";
import { getDefaultLambdaEnvironment, getStackLambdaName } from "./constructs/node-lambda";

const api = new RestServerlessApi(this, "Intake", {
    name: `ClientIntake-${props.deploymentStage}`,
    stageName: props.deploymentStage.toLowerCase(),
    corsOrigin: process.env.CORS_ORIGIN || "*",
    logging: {
        enabled: true,
        deploymentStage: props.deploymentStage,
        notificationEmail: process.env.ALERT_EMAIL || undefined,
    },
});

const lambdaName = (name: string): string => getStackLambdaName(name, props.deploymentStage);
const handlerPath = (filePath: string): string => path.join(__dirname, "..", filePath);
const cognitoAuthorizer = api.createCognitoAuthorizer(userPool);

api.setDefaultRouteOptions({
    authorizer: cognitoAuthorizer,
    environmentVariables: getDefaultLambdaEnvironment(),
});

api.get({
    routePath: "/clients",
    lambdaName: lambdaName("ClientsQuery"),
    filePath: handlerPath("src/handlers/client.handler.ts"),
    handlerName: "queryClientHandler",
    description: "/clients",
    scopes: [readAuthScope],
});
```

## Guidance

- If the repository already has centralized route registration, extend it.
- If it does not, introduce one reusable stack or construct layer instead of scattering `addMethod` calls.
- Prefer shared route helpers such as `get`, `getById`, `post`, `put`, and `delete` over raw `addMethod` once the pattern exists.
- `routePath` must start with `/`.
- Shared environment variables belong in one common helper or route-defaults path, not duplicated inline on every route.
- Persistence is SQL, so do not add table grants or DynamoDB-specific route options.
