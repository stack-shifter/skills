# REST API Pattern

Use this reference when you need a compact reminder of how this repository composes API Gateway REST routes through its local constructs.

Do not use this reference to replace local code discovery. Read `lib/core-stack.ts`, `lib/constructs/rest-api.ts`, and `lib/constructs/api-models.ts` first, then use this file as a summary.

## Goal

Add or modify REST endpoints in the way this project already does it.

## Repository Shape

- API Gateway REST API via `RestServerlessApi`
- one Lambda handler export per route
- shared Cognito authorizer set with `setDefaultRouteOptions(...)`
- per-route Cognito scopes passed from `lib/core-stack.ts`
- route files wired with `handlerPath(...)` and `lambdaName(...)`

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

- Keep route registration in `lib/core-stack.ts` unless the repository grows a new stack boundary.
- Prefer the existing `get`, `getById`, `post`, `put`, and `delete` helpers over raw `addMethod`.
- `routePath` must start with `/`.
- Shared environment variables belong in `getDefaultLambdaEnvironment()`, not duplicated inline on every route.
- Persistence is SQL, so do not add table grants or DynamoDB-specific route options.
